# Adding rudimentary support for an IsoPdom Simulation for the Gen2 Upgrade

## Requires
1. import dashi module
2. go to ~/.local/lib/python3.10/site-packages/dashi/odict.py and change
`from collections import ..` to `from collections.abc import ..` <br />
##### Running the `singletable_quality.py` script
3. import termcolors using `pip install termcolor`
4. install tqdm using `pip install tqdm`

## A little background 

The new generation of mDOMs aims to detect photons coming from different directions
A very simple assumption that can somehow shed some light into the expected behaviour
is assuming that the detector is a sphere and can detect photons coming from all the possible
angles. Since the currently deployed pDOMS have their detectors placed in their lower half 
this work focuses on  _creating_ an upper part detector and on combining them in order to have a spherical one.
Details about the implementation and the assumptions made are specified later. 


## The added changes

The first step for creating the tables is running the `singletable_tabulator_batch.py` script.
Below is the list of options which is provided and supported when running the script:

```
parser.add_option("--seed", dest="seed", type="int", default=None, help="Seed for random number generators; harvested from /dev/random if unspecified.")
parser.add_option("--nevents", dest="nevents", type="int", default=100, help="Number of light sources to inject [%default]")
parser.add_option("--z", dest="z", type="float", default=0., help="IceCube z-coordinate of light source, in meters [%default]")
parser.add_option("--zenith", dest="zenith", type="float", default=0., help="Zenith angle of source, in IceCube convention and degrees [%default]")
parser.add_option("--azimuth", dest="azimuth", type="float", default=0., help="Azimuth angle of source, in IceCube convention and degrees [%default]")
parser.add_option("--energy", dest="energy", type="float", default=1, help="Energy of light source, in GeV [%default]")
parser.add_option("--light-source", choices=('cascade', 'flasher', 'infinite-muon'), default='cascade', help="Type of light source. If 'infinite-muon', Z will be ignored, and tracks sampled over all depths. [%default]")
parser.add_option("--flasherwidth", dest="flasherwidth", type="float", default=127, help="Width of the flasher source [%default]")
parser.add_option("--flasherbrightness", dest="flasherbrightness", type="float", default=127, help="Brightness of the flasher source [%default]")
parser.add_option("--icemodel", choices=('spice_mie', 'spice_lea_full', 'spice_lea_flat', 'spice_lea_tilt', 'spice_lea_anisotropy', 'spice_3.2.1_full', 'spice_3.2.1_flat', 'spice_3.2.1_exagg', 'spice_3.2.1_noexagg', 'spice_3.2.2_full', 'spice_3.2.2_flat', 'spice_bfr-v2_flat', 'spice_bfr-v2_full', 'spice_bfr-v2_anisotropy'), default='spice_bfr-v2_flat', help="Ice model [%default]")
parser.add_option("--holeice", choices=('as.h2-50cm', 'as.flasher_p1_0.30_p2_0', 'as.flasher_p1_0.35_p2_0', 'as.nominal'), default='as.h2-50cm', help="DOM angular sensitivity model [%default]")
parser.add_option("--tablesize", choices=('half', 'full'), default='half', help="Table size in azimuth extension [%default]")
parser.add_option("--tabulate-impact-angle", default=False, action="store_true", help="Tabulate the impact angle on the DOM instead of weighting by the angular acceptance")
parser.add_option("--prescale", dest="prescale", type="float", default=1, help="Only propagate 1/PRESCALE of photons. This is useful for controlling how many photons are simulated per source, e.g. for infinite muons where multiple trajectories need to be sampled [%default]")
parser.add_option("--step", dest="steplength", type="float", default=1, help="Sampling step length in meters [%default]")
parser.add_option("--overwrite", dest="overwrite", action="store_true", default=False, help="Overwrite output file if it already exists [%default]")
parser.add_option("--errors", dest="errors", action="store_true", default=False, help="Write the errors to the fits table in addition to values [%default]")
parser.add_option("--extend", dest="extend", action="store_true", default=False, help="Include cascade extension [%default]")
parser.add_option("--rrange", dest="rrange", type="float", default=1000., help="maximum radial range for binning of table")
parser.add_option("--rbins", dest="rbins", type="int", default=283, help="number of radial bins")
parser.add_option("--tbins", dest="tbins", type="int", default=105, help="number of time bins")
parser.add_option("--mergeonly", action="store_true", default=False, dest="mergeonly", help="just merge parallel files")
parser.add_option("--verbose", action="store_true", default=False, dest="verbose", help="Enable verbose logging")
```

###### The options are self explanatory but there are some things to point out:

- Defining the `--seed` parameter is important to have an even photon distribution 
- In order to create a *.fits* file with 10k events , 100 batches of 100 events each with differnt `--seed` parameters were needed(_since the photon propagation is an embarrassingly parallel[[3]](https://en.wikipedia.org/wiki/Embarrassingly_parallel) process, slicing in batches and then merging them is quicker_)
- For the production of the flat model table, the default ice model is used (**spice_bfr-v2_flat**)
- In order to also add the `effective distance` the tablesize should be set to `'full'` (_not yet implemented_)

##### Additions in tabularor.py

The basic idea was to use the already existing models, with modifications in order to simulate an isopdom.
- At the end of the `singletable_tabulator(_batch).py`: <br />
    `tray.AddSegment(TabulatePhotonsFromSource...)` <br />
    `TabulatePhotonsFromSource` is a module imported from `tabulator.py`. Since this is not a part of the icecube software, `tabulator_batch.py` script is the one with the additions for the isopdom. <br />
    This module defines among **many** other things:
    ### ELABORATE ON EACH ONE
    - The DOM Radius
    - The reference Area
    - The DOM Acceptance (Wavelength Acceptance)
    - The Angular Sensitivity/Angular Acceptance

- 3 more options regarding the parameter `--sensor` have been added
  - lowerhalf
  - upperhalf
  - isotropic
(_lower and upper halves can be combined to produce the isotropic one, the option was added for completeness_)

- Since the DEgg's properties are closer to what is to be simulated we used its radius and reference area and altered the **angular sensitivity** and **wavelength acceptance** in a way that:
   - We needed the product <br />
     ${angSensitivity\*referenceAreaDegg\*wavelengthAcceptanceDegg(400nm) = 100cm<sup>2</sup>}$ <br />
     since these are the measurments for the mDOM. <br />
     ###### Angular Sensitivity
     The angular sensitivity function is a polynomial $P(\cos\theta)$ of 11-th order and is defined by its coefficients<br />
     (more details can be found under the original _GetAngularSensitivity_ functions)[[4]](https://github.com/icecube/icetray/tree/main/clsim/python) <br />
     
     The idea is to use a simple geometric representation of the DOM's angular sensitivity which is defined by : <br />
     ${1 \over 2}{(1 \pm \cos\theta)}$ <br />
     where $\theta$ is the direction of the photon, $+$ is used for the `lowerhalf` type sensor ( $\theta = 0$ when the photon arriving from the conventionally used `-inf`) and the function
     defined gives us the cross section area of the sphere. <br />
     The added function used to incorporate this geometry is called **GetGeometricAngularSensitivity** and is found in the `tabulator_batch.py` script. The result is a **I3CLSimFunctionPolynomial** and the first element of the array given is the **a<sub>0</sub>** (constant)coefficient, whereas the last one is the **a<sub>n-1</sub>**  
     The function is provided below : <br />
     ```
     def GetGeometricAngularSensitivity(type='lowerhalf'):
        import numpy as np
        if type == 'lowerhalf':
            coeffs=np.array([1./2,1./2,0.,0.,0.,0.,0.,0.,0.,0.,0.])
        elif type=='upperhalf':
            coeffs = np.array([1./2,-1./2,0.,0.,0.,0.,0.,0.,0.,0.,0.])
        elif type=='isotropic':
            coeffs = np.array([1.,0.,0.,0.,0.,0.,0.,0.,0.,0.,0.])
        return I3CLSimFunctionPolynomial(coeffs)
     ```
   - In order to satisfy the first bullet point of this sublist (product=100cm<sup>2</sup>) the option **active_fraction** was set to $1.2265621103449313$. The calculation behind it was straightforward and can be found in the `plots.ipynb` file.
   ###### use _I3Units.(mm,nanometer,m,etc.)_ when querying 
   ###### _Get[insert type of sensor]Acceptance.GetValue()_ returns the wanted acceptace
   ###### np.vectorize() was used on the _GetValues_ attribute
   ###### the result of the product is in m<sup>2</sup>

    

    


