<!-- BEGIN COMMENT -->

[<< Previous Chapter](CMAQ_UG_ch05_compile_and_run.md) - [Home](README.md) - [Next Chapter >>](CMAQ_UG_ch07_model_outputs.md)

<!-- END COMMENT -->

# 6. Model Configuration Options

## 6.1 Introduction
As discussed in [Chapter 1](CMAQ_UG_ch01_overview.md), CMAQ is a multipollutant, multiscale air quality modeling system that estimates the transport and chemistry of ozone, PM, toxic airborne pollutants, and acidic and nutrient pollutant species, as well as visibility degradation and deposition totals. CMAQ includes state-of-the-art technical and computational techniques to simulate air quality from urban to global scales. It can model complex atmospheric processes affecting transformation, transport, and deposition of air pollutants using a system architecture that is designed for fast and efficient computing. This chapter presents a brief overview of the conceptual formulation of Eulerian air quality modeling and the science features in various components of the Chemistry-Transport Model (CTM) component of CMAQ, CCTM. 

## 6.2 Numerical Approach
The theoretical basis for CMAQ’s formulation is the conservation of mass for atmospheric trace species emissions, transport, chemistry, and removal in the atmosphere. The general form of a chemical species equation derives from this conservation, so that changes in atmospheric concentrations of a species, C<sub>i</sub>, can mathematically be represented as

![Equation 8-1](images/Figure8-1.JPG)  

where the terms on the right-hand side of the equation represent the rate of change in C<sub>i</sub> due to advection, turbulent mixing, cloud processes (mixing, scavenging, and aqueous-phase chemistry), dry deposition, and aerosol processes (phase partitioning, and aerosol dynamics). R<sub>gi</sub> represents the rate of change due to gas and heterogeneous chemical reactions, while E<sub>i</sub> is the emission rate for that species. The mass conservation for trace species and the moment dynamic equations for the various modes of the particulate size distribution in CMAQ are further formulated in generalized coordinates, where in the same formulation allows the model to accommodate the commonly used horizontal map projections (i.e., Lambert conformal, polar stereographic, and Mercator) as well as different vertical coordinates (see Chapters 5 and 6 in Byun and Ching, 1999). The governing equation for CMAQ is numerically solved using the time-splitting or process splitting approach wherein each process equation is solved sequentially, typically with the process with the largest time-scale solved first. 

## 6.3 Grid Configuration
CMAQ is a three-dimensional Eulerian air quality model. To solve the governing partial differential equations, the domain of a model run (the volume of the atmosphere over a geographic region) is discretized with three-dimensional cells. The grid cells and boundaries of domain must be rigorously and consistently defined for all functional components of the model (e.g., chemistry, emissions, meteorology). 
CMAQ’s generalized coordinate formulation maps the physical space to the computational space. The horizontal grid specification (setting the x and y dimensions) must be regular: the horizontal projection of each grid cell (sometimes referred to as a pixel) has the same resolution, and the boundaries of each pixel are time-invariant. By contrast, the vertical grid specification (setting the z dimension) need not be regular.

After determining the horizontal and vertical extent of the domain of interest, a meteorological model must be run for a horizontal domain slightly larger than the CMAQ domain. A larger meteorology domain is necessary for distinguishing the meteorological boundary conditions from the CMAQ boundary conditions.

Available horizontal grids for a given CMAQ run are defined at runtime by setting the GRIDDESC and GRID_NAME environment variables to point to an existing grid definition file and to one of the grids defined in the file, respectively. Horizontal grids are defined by the
grid definition file (GRIDDESC), which can be edited by the user.  Further details on grid configuration are available in the [README.md](../../PREP/mcip/README.md) file in the PREP/mcip folder.

## 6.4 Science Configurations
CCTM contains several science configurations for simulating transport, chemistry, and deposition. All of the science configuration options in CCTM, such as the chemical mechanism to be used, are set when compiling the executable. The model grid and vertical layer structure for CCTM are set at execution. The important distinction between selecting the science configuration and the model grid/layer configuration is that CCTM does not need to be recompiled when changing model grids/layers but does need to be recompiled when new science options are invoked.  The following sections describe how these science options can be utilized by configuring using the `bldit_cctm.csh` and `run_cctm.csh` scripts.  For the remainder of this chapter these files will be referred to as simply BuildScript and RunScript.

**>>COMMENT<<** Each of the following sections should provide a brief description of the science process followed by the specific configuration options that need to be set in both the build script (BuildScript) and run script (RunScript). The information on these environment variables should be consistent with what is provided in Appendix A.  When specifying environment variable options please use the following syntax:
```
setenv ModDepv depv/m3dry
```

## 6.5 Advection
In CCTM, the 3-dimensional transport by mean winds (or advection) is numerically represented by sequentially solving locally-one dimensional equations for the two horizontal and vertical components. CMAQ uses the piecewise parabolic method (PPM) (Colella and Woodward, 1984) for representing tracer advection in each of the three directions. This algorithm is based on the finite-volume sub-grid definition of the advected scalar. In PPM, the sub-grid distribution is described by a parabola in each grid interval. PPM is a monotonic and positive-definite scheme. Positive-definite schemes maintain the sign of input values, which in this case means that positive concentrations will remain positive and cannot become negative.

Mass consistency is a key desired attribute in tracer advection. Data consistency is maintained for air quality simulations by using dynamically and thermodynamically consistent meteorology data from WRF/MCIP. Mass inconsistencies can nevertheless arise either through the use of different grid configurations (horizontal or vertical) or due to differing numerical advection schemes between the driving meteorological model and the CCTM. While inconsistencies due to the former can be eliminated through use of the same grid configurations (thus, layer collapsing is not recommended), some inconsistencies can still remain due to differing numerical representations for satisfying the mass-continuity equation between the driving meteorological model and the CCTM. These mass-inconsistencies manifest as first order terms (whose magnitude can often be comparable to tracer lifetimes if continuity is not satisfied with high accuracy) that can artificially produce or destroy mass during 3D tracer advection (e.g., Mathur and Peters, 1990).

CMAQ has two options that minimize mass consistency errors in tracer advection. In one scheme, that was first implemented in CMAQv4.5 and later improved for CMAQv4.7.1, designated “yamo” in the BLD  script, CMAQ  advects air density and re-diagnoses the vertical velocity field according to the layer-by-layer mass continuity equation which guarantees that the CCTM advected density matches that derived from the driving meteorological inputs (e.g., Odman and Russell, 2000). Briefly, x- and y-advection are first performed (the order of these is reversed every step to minimize aliasing errors) to yield intermediate tracer and density fields. The intermediate density field is then subject to vertical advection with the PPM scheme such that it yields the WRF derived density field at the end of the advection time-step. This scheme results in an estimated vertical velocity field that is minimally adjusted relative to the WRF derived field in the lower model layers but yields strict mass-consistent tracer advection in CMAQ.  A drawback to this approach is that erroneous noise in the diagnosed vertical velocity field accumulates toward the top of the model with non-zero velocity and mass flux across the top boundary.  The noise in the vertical velocity field causes excessive diffusion in upper layers.  Therefore, starting in CMAQv5.0, a new scheme was implemented, designated “wrf”, that closely follows the vertical velocity calculation in WRF.  This scheme solves the vertically integrated mass continuity equation such that the collunm integrated horizontal mass divergence is balanced by the net change in column mass (Skamarock et al, 2019).  An advantage of this scheme is that the diagnosed vertical velocity agrees more closely with the WRF vertical velocity field with zero velocity and mass flux across the upper model boundary.  Thus, the spurious velocity noise and excessive diffusion in the upper layer are eliminated.  The main drawback of this scheme is that mass conservation is not guaranteed so density must be updated from the meteorology inputs every timestep.  The “wrf” option is the recommended configuration for CMAQv5.3.

## 6.6 Horizontal Diffusion
## 6.7 Vertical Diffusion
### 6.7.1 Dry Deposition - m3dry
### 6.7.2 Dry Depostion - STAGE
## 6.8 Emissions

CMAQ introduces emissions of trace gases and aerosols from a variety of important sources (e.g. electric generating utilities, vehicles, fires, trees, dust storms, farms, etc.). Some emissions are applied in the surface layer of the model grid, while others are applied at higher altitudes if, for example, they originate from point source like an elevated stack, or a large forest fire. Many sources that are related to local meteorology may be calculated online in CMAQ. However, most sources, especially anthropogenic ones, are preprocessed using software like the Sparse Matrix Operator Kerner Emissions (SMOKE) Modeling System. Once these external tools have calculated the offline emissions, they may merge them into larger aggregated files. We refer to emissions that are either calculated online or read into CMAQ from a file as emission "streams".

Because CMAQ represents both primary and secondary pollutants, emissions are processed for a subset of the species CMAQ treats. The emissions chemical speciation must be compatible with the chemical mechanism chosen for CMAQ (e.g. cb6r3_ae7_aq) because different mechanisms represent large compounds like functionalized hydrocarbons with different formulae. CMAQv5.3 has introduced new features that make the process of mapping emissions species to CMAQ species more transparent and flexible (see Emission Control with DESID[link]). In fact, users can now toggle, modify, and augment emissions from all available streams in order to better tailor their simulations to the questions they are asking CMAQ to help answer. For tutorials covering specific tasks, please see the DESID tutorial page [link].

### 6.8.1 Emission Streams
Depending on the nature of any stream and the information used to quantify its emissions, it may be treated as one of three types:

#### Online Stream:
CMAQ will calculate the emission rates from this source using information about local meteorology, land characteristics, etc. The streams available for running Online in CMAQ are: biogenics (BEIS) [link], marine gas[link], lightning NO [link], wind-blown dust [link], and sea-spray[ link].

#### Gridded Stream (offline):
CMAQ will read emission rates from an input file, which is organized into an array that is identical in shape to the grid CMAQ is running. Typically these rates are stored at hourly time points and are then interpolated within CMAQ to each time step. Some common examples of Gridded emissions include:

- Mobile sources such as passenger vehicles, trains, ships, scooters, etc.
- Low-level point source emissions that are not large enough to be treated individually
- Residential heating
- Consumer product use (e.g. adhesives, personal care products, pesticides, etc.)
- Agricultural (e.g. burning, dust, animal waste, etc.)
- Road, Construction and mechanically generated dust
- Biogenic VOCs (if not calculated online with BEIS)

Users add Gridded emissions to a simulation via the RunScript. First the variable N_EMIS_GR must be set to the number of Gridded Streams to be used:

```
setenv N_EMIS_GR 3
```

The RunScript must also specify the location of the input files using three-digit suffixes for the stream number:

```
setenv GR_EMIS_001 /home/user/path-to-file/emiss_stream_1_${DATE}.nc
```

and the short-name label to be used to refer to the Stream in logfiles:

```
setenv GR_EMIS_LAB_001 MOBILE
```
If N_EMIS_GR is set 0, then CMAQ will run with no Gridded emissions even if the values for GR_EMIS_XXX and GR_EMIS_LAB_XXX are all set.

#### Inline Stream (offline):
For these streams, emission rates and stack characteristics are provided for many individual sources on the same file. CMAQ uses the stack information to calculate important quantities like the injection height online taking into account local meteorology. A specific latitude/longitude pair is given for each source to locate it in the CMAQ grid.  Some common examples of Inline emissions include:

- Stacks (electric generation units, industrial sources, manufacturing, etc.)
- Forest fires
- Large prescribed fire events  

Users add Inline emissions to a simulation via the RunScript. First the variable N_EMIS_PT must be set to the number of Inline Streams to be used:

```
setenv N_EMIS_PT 3
```
The RunScript must also specify the location of the input files using three-digit suffixes for the stream number:

```
setenv STK_EMIS_002 /home/user/path-to-file/inline_emiss_stream_2_${DATE}.nc
```

The location to the "stack file" with static information about the properties of each source on the stream:
```
setenv STK_GRPS_002 /home/user/path-to-file/inline_stack_groups_2.nc
```
and the short-name label to be used to refer to the Stream in logfiles:
```
setenv STK_EMIS_LAB_002 POINT_FIRES
```
If N_EMIS_PT is set 0, then CMAQ will run with no Inline emissions even if the values for STK_EMIS_XXX, STK_GRPS_XXX and STK_EMIS_LAB_XXX are all set.

### 6.8.2 Online Emission Streams
#### BEIS
#### Plume Rise 
#### Wind-Blown Dust
#### Sea Spray
#### Lightning NO

## 6.9 Gas Phase Chemistry
### 6.9.1 Gas Phase Chemical Mechanisms
### 6.9.2 Solvers
### 6.9.3 Photolysis

## 6.10 Aerosol Dynamics and Chemistry
### 6.10.1 aero6, aero7

## 6.11 Aqueous-Phase Reaction and Cloud Mixing
### 6.11.1 AQCHEM
### 6.11.2 AQCHEM-KMT
### 6.11.3 AQCHEM-KMTI

## 6.12 Potential Vorticity Scaling

## 6.13 Coupled WRF-CMAQ 

## References
Colella, P., and P. L. Woodward, 1984: The piecewise parabolic method (PPM) for gas-dynamical simulations. J. Comput. Phys., 54, 174–201.

Mathur, R. and L.K. Peters, 1990: Adjustment of wind fields for application in air pollution modeling, Atmos. Environ., 24(5), 1095-1106.

Odman, M. T., and A. G. Russell, 2000: Mass conservative coupling of non-hydrostatic meteorological models with air quality models, in Air Pollution Modelling and Its Application XIII, edited by S.-E. Gryning and E. Batchvarova, pp. 651-660, Kluwer Academic/Plenum Publishers, New York.

Skamarock, W.C., Klemp, J.B., Dudhia, J., Gill, D.O., Liu, Z., Berner, J., Wang, W., Powers, J.G., Duda, M.G., Barker, D.M., Huang, X.Y., A description of the advanced research WRF version 4. NCAR TECHNICAL NOTE, NCAR/TN–556+STR, March, 2019.


<!-- BEGIN COMMENT -->

[<< Previous Chapter](CMAQ_UG_ch05_compile_and_run.md) - [Home](README.md) - [Next Chapter >>](CMAQ_UG_ch07_model_outputs.md)
<br>
CMAQ User's Guide (c) 2019<br>

<!-- END COMMENT -->