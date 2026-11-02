[README.md](https://github.com/user-attachments/files/34268888/README.md)
# Lagrangian particle tracking in the Venice Lagoon

This test extends the [three-dimensional Venice Lagoon hydrodynamic simulation](https://github.com/georgu/shyfemcm-ismar/wiki/%231-Three%E2%80%90dimensional-Venice-lagoon-simulation) by activating the SHYFEM Lagrangian particle-tracking module. The hydrodynamic model and the particle model run together: the circulation is computed while particles are released, transported and written to a separate output file.

The lagoon geometry, vertical layers, tidal boundaries and atmospheric forcing are those of the hydrodynamic example. Here we concentrate on the additional particle settings, running the combined simulation, plotting particle positions and creating an animation.

In this example, particles are released across the model domain at hourly intervals during the first day. Tracking continues until the end of the second day, allowing us to follow the previously released particles after releases stop. The experiment illustrates particle transport under idealized lagoon conditions; the particles have no prescribed mass or pollutant concentration.

# Compilation options

Use the compilation setup described under [Compilation options](https://github.com/georgu/shyfemcm-ismar/wiki/%231-Three%E2%80%90dimensional-Venice-lagoon-simulation#compilation-options) in the hydrodynamic tutorial. Run this test with the serial executable.

The animation step additionally requires Ghostscript and FFmpeg. On Ubuntu, these can be installed with:

```bash
sudo apt install ghostscript ffmpeg
```

They are only needed for converting the plots into an animation.

# Test directory and data

Follow [Test directory and data](https://github.com/georgu/shyfemcm-ismar/wiki/%231-Three%E2%80%90dimensional-Venice-lagoon-simulation#test-directory-and-data) in the hydrodynamic tutorial for the mesh, forcing files and executable setup. Work in the directory containing `GRID/` and `INPUT/`.

The additional files used here are:

| File | Purpose |
|---|---|
| `venlag_hydro_lgr.str` | Simulation configuration containing the hydrodynamic setup and the additional `$lagrg` section. |
| `apn_venice_lagoon_lgr.str` | Plotting configuration for particle positions. |
| `animate_particles.sh` | Conversion of the PostScript plots into MP4 and GIF animations. |

All commands below are run from this test directory. If you use the supplied archive, make sure the `shyfem` and `shyplot` executable links point to your own installation.

# The configuration file

The [hydrodynamic configuration](https://github.com/georgu/shyfemcm-ismar/wiki/%231-Three%E2%80%90dimensional-Venice-lagoon-simulation#the-configuration-file) explains the existing sections. Keep the hydrodynamic settings in the supplied `venlag_hydro_lgr.str`, including the forcing filenames in `$name`.

For this experiment, the hydrodynamic simulation covers two complete days. Check these entries in `$para`:

```text
        date   = '2014-08-01'   time  = 0
        itanf  = '2014-08-01'   itend = '2014-08-03'
```

A date without a time denotes midnight. Thus, the simulation starts on 1 August 2014 at 00:00 and ends on 3 August at 00:00.

### The section `$lagrg`

Add the following section to the simulation configuration, outside any other section. In the supplied file it is placed after the open-boundary sections:

```text
$lagrg
        ilagr = 3

        itlanf = '2014-08-01'  itlend = '2014-08-03'
        itranf = '2014-08-01'  itrend = '2014-08-02'
        itmlgr = '2014-08-01'  idtlgr = '10m'
        idtl = '1h'

        nbdy = 100
        ipvert = 0
        nbdymax = 10000000
        rwhpar = 0.2

        stkpar = 0.01  dripar = 1.0
        lintop = 1
        lbeach = 0
$end
```

#### Activating particle tracking

`ilagr` selects the particle-tracking mode:

| Value | Mode |
|---|---|
| `0` | Particle tracking disabled. |
| `1` | Surface particle tracking. |
| `2` | Two-dimensional tracking, without vertical advection and diffusion. |
| `3` | Three-dimensional particle tracking. |

Here, `ilagr = 3` enables three-dimensional transport. The particle model uses the circulation calculated during the same run, so no separate offline particle simulation or conversion of the hydrodynamic output is required.

#### Tracking, release and output times

There are three separate schedules:

| Parameters | Function | Settings in this example |
|---|---|---|
| `itlanf`, `itlend` | Start and end of particle tracking. | 1 August 00:00 to 3 August 00:00. |
| `itranf`, `itrend`, `idtl` | Start, end and interval of particle releases. | Hourly releases from 1 August 00:00 through the release window ending on 2 August 00:00. |
| `itmlgr`, `idtlgr` | Start and interval of particle output. | Output from 1 August 00:00, every 10 minutes. |

After the release window closes, existing particles continue to move until `itlend`. The final tracking time must fall within the hydrodynamic simulation period.

`idtl` is the release interval, and `idtlgr` controls how often positions are saved. The hydrodynamic time-stepping settings are explained in the original tutorial.

For a single initial release, set `idtl = 0`; a negative value disables these scheduled releases. Setting `idtlgr = 0` disables particle output.

#### Number and initial distribution of particles

`nbdy = 100` specifies the nominal number of particles for each release. No release-area file (`lgrlin`) or trajectory-based release file (`lgrtrj`) is supplied, so the module seeds particles across the whole computational domain.

For this domain-wide release method, SHYFEM derives a horizontal spacing from the domain area and `nbdy`, then inserts particles at suitable locations inside the mesh. Consequently, `nbdy` controls the seeding density; the actual number inserted can differ from exactly 100. The run reports the inserted particle count.

`nbdymax = 10000000` sets the upper limit on the number of particles handled by the module. It is a capacity limit; therefore, it must be large enough for the particles accumulated during repeated releases.

`ipvert = 0` inserts particles in the surface layer, at its relative mid-depth. It controls their initial vertical distribution. With `ilagr = 3`, particles can subsequently move vertically through advection and turbulent diffusion: this configuration does not keep them permanently at the surface.

`lintop = 1` specifies the upper layer for vertically distributed releases. With the present `ipvert = 0`, insertion goes directly into layer 1, so changing `lintop` alone will not move the initial release to a different depth. For releases distributed through the water column, `ipvert` and the vertical release limits need to be configured together.

#### Diffusion, additional drift and beaching

| Parameter | Meaning in this example |
|---|---|
| `rwhpar = 0.2` | Constant horizontal particle diffusivity of 0.2 m²/s, represented with a random walk. It spreads trajectories in addition to resolved advection. |
| `stkpar = 0.01` | Adds a wind-proportional drift to particles in layer 1. In this implementation the added velocity is 0.01 times the local wind velocity. |
| `dripar = 1.0` | Leaves the advective transport unscaled. This parameter is provided for calibrating drifter response. |
| `lbeach = 0` | Disables the optional shoreline-beaching parameterization. |

The manual describes `stkpar` as a parameterization of wind drag and Stokes drift. Here it is implemented using the local wind components, so `0.01` corresponds to a drift of 1% of the wind velocity while the particle is in the surface layer. Setting `stkpar = 0` removes this additional particle drift; wind still acts on the hydrodynamic circulation through the hydrodynamic forcing.

# Run and plotting

### Running the simulation

Run the combined hydrodynamic and Lagrangian configuration:

```bash
./shyfem venlag_hydro_lgr.str
```

Wait for the run to finish and check the terminal output for errors. Since the short simulation name in `$title` is `venlag`, the particle output is:

```text
venlag.lgr.shy
```

The usual hydrodynamic and temperature/salinity outputs, `venlag.hydro.shy` and `venlag.ts.shy`, are also produced. Their postprocessing is described in [Run and plotting](https://github.com/georgu/shyfemcm-ismar/wiki/%231-Three%E2%80%90dimensional-Venice-lagoon-simulation#run-and-plotting) of the hydrodynamic tutorial.

### Plotting particle positions

The particle plotting file retains the Venice Lagoon map settings from `apn_venice_lagoon.str`. The relevant additions in `apn_venice_lagoon_lgr.str` are the following entries **inside the existing `$para` section**:

```text
        lgrtrj = 0
        lgmean = 0
```

`lgrtrj = 0` selects particle positions rather than trajectory lines. `lgmean = 0` selects individual particles rather than group-mean positions.

Add a separate colour section:

```text
$color lagrangian
        legcol = 'Particles'
        lgrtyp = 0
        lgrcol = 0.
        icolor = 0
        ipllog = 0
$end
```

`lgrtyp = 0` displays all particles with the same colour, selected by `lgrcol`. This gives a simple view of movement and spreading. The alternative `lgrtyp = 1` colours particles by age and requires an appropriate colour scale.

Create the plots with:

```bash
./shyplot venlag.lgr.shy apn_venice_lagoon_lgr.str
```

`shyplot` writes a multipage PostScript file named `plot.ps`, with a page for each plotted output record. An existing `plot.ps` is overwritten. For the supplied completed run, the file contains 289 frames spanning the two-day simulation at 10-minute intervals.

To reduce the number of frames, plot every sixth saved record:

```bash
./shyplot -freq 6 venlag.lgr.shy apn_venice_lagoon_lgr.str
```

Here, that gives approximately hourly frames. Use the full output sequence for the smoother animation below.

### Creating an animation

`shyplot` creates the sequence of plots. The supplied `animate_particles.sh` converts it into animations using Ghostscript and FFmpeg:

```bash
bash animate_particles.sh plot.ps particle_animation
```

The first argument is the input PostScript file. The second is a new output directory; it must not already exist. For another conversion, choose a different directory name.

The script creates:

| Output | Format |
|---|---|
| `particle_animation/venice_particles.mp4` | Video for playback and sharing. |
| `particle_animation/venice_particles.gif` | Looping animation suitable for embedding in documentation. |

The animation settings near the beginning of the supplied script are:

```bash
fps=10
dpi=300
gif_width=1040
```

`fps` controls playback speed, `dpi` controls the rendering resolution and `gif_width` controls the GIF width in pixels.

At 10 frames per second, the supplied 289-frame sequence lasts about 29 seconds. Each second of playback represents 100 minutes of model time because the saved records are 10 minutes apart. Changing `fps` changes playback speed; changing `idtlgr` changes the saved temporal resolution and requires rerunning the simulation.

On Ubuntu, open the video with:

```bash
xdg-open particle_animation/venice_particles.mp4
```
![Particle movement in the Venice Lagoon]()

The animation lets us follow successive releases and their redistribution through the lagoon. Changes in the number of visible particles can reflect new releases and particles leaving the active set. A horizontal position plot alone does not show the vertical redistribution allowed by `ilagr = 3`.
