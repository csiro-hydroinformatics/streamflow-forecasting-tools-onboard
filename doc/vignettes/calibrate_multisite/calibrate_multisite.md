Calibration of a catchment using multisite multiobjective composition
================
Jean-Michel Perraud
2020-01-28

# Calibration of a catchment using multisite multiobjective composition

# Use case

This vignette demonstrates how one can calibrate a catchment using
multiple gauging points available within this catchment.

This is a joint calibration weighing multiple objectives, possibly
sourced at different modelling elements, thus a whole-of-catchment
calibration technique. Staged, cascading calibration is supported and
described in another vignette.

# Data

The sample data that comes with the package contains a model definition
for the South Esk catchment, including a short subset of the climate and
flow record data.

``` r
# swiftr_dev()   

library(swift)

modelId <- 'GR4J'
siteId <- 'South_Esk'
simulation <- sampleCatchmentModel(siteId=siteId, configId='catchment')
# simulation <- swapModel(simulation, 'MuskingumNonLinear', 'channel_routing')
simulation <- swapModel(simulation, 'LagAndRoute', 'channel_routing')
```

``` r
seClimate <- sampleSeries(siteId=siteId, varName='climate')
seFlows <- sampleSeries(siteId=siteId, varName='flow')
```

The names of the climate series is already set to the climate input
identifiers of the model simulation, so setting them as inputs is easy:

``` r
playInput(simulation, seClimate)
setSimulationSpan(simulation, start(seClimate), end(seClimate))
setSimulationTimeStep(simulation, 'hourly')
```

Moving on to define the parameters, free or fixed. We will use (for now
- may change) the package calibragem, companion to SWIFT.

``` r
configureHourlyGr4j(simulation)
```

We define a function creating a realistic feasible parameter space. This
is not the main object of this vignette, so we do not describe in
details.

``` r
createMetaParameterizer <- function(simulation, refArea=250, timeSpan=3600L) {
  timeSpan <- as.integer(timeSpan)
  parameterizer <- defineGr4jScaledParameter(refArea, timeSpan)
  
  # Let's define _S0_ and _R0_ parameters such that for each GR4J model instance, _S = S0 * x1_ and _R = R0 * x3_
  pStates <- linearParameterizer(
                      paramName=c("S0","R0"), 
                      stateName=c("S","R"), 
                      scalingVarName=c("x1","x3"),
                      minPval=c(0.0,0.0), 
                      maxPval=c(1.0,1.0), 
                      value=c(0.9,0.9), 
                      selectorType='each subarea')
  
  initParameterizer <- makeStateInitParameterizer(pStates)
  parameterizer <- concatenateParameterizers(parameterizer, initParameterizer)
  
  lagAndRouteParameterizer <- function() {
    p <- data.frame(Name = c('alpha', 'inverse_velocity'),
        Value = c(1, 1),
        Min = c(1e-3, 1e-3),
        Max = c(1e2, 1e2),
        stringsAsFactors = FALSE)
    p <- createParameterizer('Generic links', p)
    return(p)
  }

  # Lag and route has several discrete storage type modes. One way to set up the modeP
  setupStorageType <- function(simulation) {
    p <- data.frame(Name = c('storage_type'), 
        Value = 1, 
        Min = 1,
        Max = 1,
        stringsAsFactors = FALSE)
    p <- createParameterizer('Generic links', p)
    applySysConfig(p, simulation)
  }
  setupStorageType(simulation)

  # transfer reach lengths to the Lag and route model
  linkIds <- getLinkIds(simulation)
  reachLengths <- getStateValue(simulation, paste0('link.', linkIds, '.Length'))
  setStateValue(simulation, paste0('link.', linkIds, '.reach_length'), reachLengths) 

  lnrp <- lagAndRouteParameterizer()
  parameterizer <- concatenateParameterizers(parameterizer, lnrp)  
  return(parameterizer)
}
```

``` r
parameterizer <- createMetaParameterizer(simulation)
setParameterValue(parameterizer, 'asinh_x2', 0)
applySysConfig(parameterizer, simulation)
execSimulation(simulation)
```

We are now ready to enter the main topic of this vignette, setting up a
weighted multi-objective for calibration purposes.

The sample gauge data flow contains identifiers that are of course
distinct from the network node identifiers. We create a map between them
(note - this information used to be in the NodeLink file in swiftv1).

``` r
gauges <- as.character(c( 92106, 592002, 18311, 93044,    25,   181))
nodeIds <- paste0('node.', as.character( c(7,   12,   25,   30,   40,   43) ))   
names(gauges) <- nodeIds
```

First, let us try the Nash Sutcliffe efficiency, for simplicity (no
additional parameters needed). We will set up NSE calculations at two
points (nodes) in the catchments. Any model state from a link, node or
subarea could be a point for statistics calculation.

``` r
s <- GetSimulationSpan_Pkg_R(simulation)

calibrationPoints <- nodeIds[1:2]
mvids <- mkFullDataId(calibrationPoints, 'OutflowRate')

w <- c(1.0, 2.0) # weights (internally normalised to a total of 1.0)
names(w) <- mvids

statspec <- multiStatisticDefinition( 
  modelVarIds = mvids, 
  statisticIds = rep('nse', 2), 
  objectiveIds=calibrationPoints, 
  objectiveNames = paste0("NSE-", calibrationPoints), 
  starts = rep(s$Start, 2), 
  ends = rep(s$End, 2) )

observations <- list(
  seFlows[,gauges[1]],
  seFlows[,gauges[2]]
)

moe <- createMultisiteObjective(simulation, statspec, observations, w)
```

``` r
getScore(moe, parameterizer)
```

    ## $scores
    ## MultisiteObjectives 
    ##          0.03281224 
    ## 
    ## $sysconfig
    ##               Name       Min        Max     Value
    ## 1           log_x4  0.000000   2.380211 0.3054223
    ## 2           log_x1  0.000000   3.778151 0.5066903
    ## 3           log_x3  0.000000   3.000000 0.3154245
    ## 4         asinh_x2 -3.989327   3.989327 0.0000000
    ## 5               R0  0.000000   1.000000 0.9000000
    ## 6               S0  0.000000   1.000000 0.9000000
    ## 7            alpha  0.001000 100.000000 1.0000000
    ## 8 inverse_velocity  0.001000 100.000000 1.0000000

We now build a parameterizer that is suited to multisite objective
functions. Note that for now there is no effect changing the result -
just scaffloding and structural test.

``` r
p <- createParameterizer('no apply')
funcParam <- list(p, clone(p)) # for the two calib points, NSE has no param here
catchmentPzer <- parameterizer
fp <- createMultisiteObjParameterizer(funcParam, calibrationPoints, prefixes=paste0(calibrationPoints, '.'), mixFuncParam=NULL, hydroParam=catchmentPzer)
(parameterizerAsDataFrame(fp))
```

    ##               Name       Min        Max     Value
    ## 1           log_x4  0.000000   2.380211 0.3054223
    ## 2           log_x1  0.000000   3.778151 0.5066903
    ## 3           log_x3  0.000000   3.000000 0.3154245
    ## 4         asinh_x2 -3.989327   3.989327 0.0000000
    ## 5               R0  0.000000   1.000000 0.9000000
    ## 6               S0  0.000000   1.000000 0.9000000
    ## 7            alpha  0.001000 100.000000 1.0000000
    ## 8 inverse_velocity  0.001000 100.000000 1.0000000

``` r
EvaluateScoreForParameters(moe, fp)
```

    ## $scores
    ## MultisiteObjectives 
    ##          0.03281224 
    ## 
    ## $sysconfig
    ##               Name       Min        Max     Value
    ## 1           log_x4  0.000000   2.380211 0.3054223
    ## 2           log_x1  0.000000   3.778151 0.5066903
    ## 3           log_x3  0.000000   3.000000 0.3154245
    ## 4         asinh_x2 -3.989327   3.989327 0.0000000
    ## 5               R0  0.000000   1.000000 0.9000000
    ## 6               S0  0.000000   1.000000 0.9000000
    ## 7            alpha  0.001000 100.000000 1.0000000
    ## 8 inverse_velocity  0.001000 100.000000 1.0000000

We can get the value of each objective. The two NSE scores below are
negative. Note above that the composite objective is positive, i.e. the
opposite of the weighted average. This is because the composite
objective is always minimisable (as of writing anyway this is a design
choice.)

``` r
EvaluateScoresForParametersWila_Pkg_R(moe, fp)
```

    ##     node.12      node.7 
    ## -0.03819707 -0.02204260

### log-likelihood multiple objective

Now, let’s move on to use log-likelihood instead of NSE.

``` r
statspec <- multiStatisticDefinition(  
  modelVarIds = mvids, 
  statisticIds = rep('log-likelihood', 2), 
  objectiveIds=calibrationPoints, 
  objectiveNames = paste0("LL-", calibrationPoints), 
  starts = rep(s$Start, 2), 
  ends = rep(s$End, 2) )

obj <- createMultisiteObjective(simulation, statspec, observations, w)
```

For this to work we need to include parameters

``` r
maxobs <- max(observations[[1]], na.rm=TRUE)
censorThreshold <- 0.01
censopt <- 0.0
calcMAndS <- 1.0 # i.e. TRUE

#       const string LogLikelihoodKeys::KeyAugmentSimulation = "augment_simulation";
#       const string LogLikelihoodKeys::KeyExcludeAtTMinusOne = "exclude_t_min_one";
#       const string LogLikelihoodKeys::KeyCalculateModelledMAndS = "calc_mod_m_s";
#       const string LogLikelihoodKeys::KeyMParMod = "m_par_mod";
#       const string LogLikelihoodKeys::KeySParMod = "s_par_mod";

p <- createParameterizer('no apply')
# Note: exampleParameterizer is also available
addToHyperCube(p, 
          data.frame( Name=c('b','m','s','a','maxobs','ct', 'censopt', 'calc_mod_m_s'),
          Min   = c(-30, 0, 1,    -30, maxobs, censorThreshold, censopt, calcMAndS),
          Max   = c(0,   0, 1000, 1, maxobs, censorThreshold, censopt, calcMAndS),
          Value = c(-7,  0, 100,  -10, maxobs, censorThreshold, censopt, calcMAndS),
          stringsAsFactors=FALSE) )

funcParam <- list(p, clone(p)) # for the two calib points, NSE has no param here
catchmentPzer <- parameterizer
fp <- createMultisiteObjParameterizer(funcParam, calibrationPoints, prefixes=paste0(calibrationPoints, '.'), mixFuncParam=NULL, hydroParam=catchmentPzer)
(parameterizerAsDataFrame(fp))
```

    ##                    Name        Min         Max       Value
    ## 1                log_x4   0.000000    2.380211   0.3054223
    ## 2                log_x1   0.000000    3.778151   0.5066903
    ## 3                log_x3   0.000000    3.000000   0.3154245
    ## 4              asinh_x2  -3.989327    3.989327   0.0000000
    ## 5                    R0   0.000000    1.000000   0.9000000
    ## 6                    S0   0.000000    1.000000   0.9000000
    ## 7                 alpha   0.001000  100.000000   1.0000000
    ## 8      inverse_velocity   0.001000  100.000000   1.0000000
    ## 9              node.7.b -30.000000    0.000000  -7.0000000
    ## 10             node.7.m   0.000000    0.000000   0.0000000
    ## 11             node.7.s   1.000000 1000.000000 100.0000000
    ## 12             node.7.a -30.000000    1.000000 -10.0000000
    ## 13        node.7.maxobs  22.000000   22.000000  22.0000000
    ## 14            node.7.ct   0.010000    0.010000   0.0100000
    ## 15       node.7.censopt   0.000000    0.000000   0.0000000
    ## 16  node.7.calc_mod_m_s   1.000000    1.000000   1.0000000
    ## 17            node.12.b -30.000000    0.000000  -7.0000000
    ## 18            node.12.m   0.000000    0.000000   0.0000000
    ## 19            node.12.s   1.000000 1000.000000 100.0000000
    ## 20            node.12.a -30.000000    1.000000 -10.0000000
    ## 21       node.12.maxobs  22.000000   22.000000  22.0000000
    ## 22           node.12.ct   0.010000    0.010000   0.0100000
    ## 23      node.12.censopt   0.000000    0.000000   0.0000000
    ## 24 node.12.calc_mod_m_s   1.000000    1.000000   1.0000000

``` r
moe <- obj
```

``` r
getScore(moe, fp)
```

    ## $scores
    ## MultisiteObjectives 
    ##             44779.5 
    ## 
    ## $sysconfig
    ##                    Name        Min         Max       Value
    ## 1                log_x4   0.000000    2.380211   0.3054223
    ## 2                log_x1   0.000000    3.778151   0.5066903
    ## 3                log_x3   0.000000    3.000000   0.3154245
    ## 4              asinh_x2  -3.989327    3.989327   0.0000000
    ## 5                    R0   0.000000    1.000000   0.9000000
    ## 6                    S0   0.000000    1.000000   0.9000000
    ## 7                 alpha   0.001000  100.000000   1.0000000
    ## 8      inverse_velocity   0.001000  100.000000   1.0000000
    ## 9              node.7.b -30.000000    0.000000  -7.0000000
    ## 10             node.7.m   0.000000    0.000000   0.0000000
    ## 11             node.7.s   1.000000 1000.000000 100.0000000
    ## 12             node.7.a -30.000000    1.000000 -10.0000000
    ## 13        node.7.maxobs  22.000000   22.000000  22.0000000
    ## 14            node.7.ct   0.010000    0.010000   0.0100000
    ## 15       node.7.censopt   0.000000    0.000000   0.0000000
    ## 16  node.7.calc_mod_m_s   1.000000    1.000000   1.0000000
    ## 17            node.12.b -30.000000    0.000000  -7.0000000
    ## 18            node.12.m   0.000000    0.000000   0.0000000
    ## 19            node.12.s   1.000000 1000.000000 100.0000000
    ## 20            node.12.a -30.000000    1.000000 -10.0000000
    ## 21       node.12.maxobs  22.000000   22.000000  22.0000000
    ## 22           node.12.ct   0.010000    0.010000   0.0100000
    ## 23      node.12.censopt   0.000000    0.000000   0.0000000
    ## 24 node.12.calc_mod_m_s   1.000000    1.000000   1.0000000

``` r
EvaluateScoresForParametersWila_Pkg_R(moe, fp)
```

    ##   node.12    node.7 
    ## -44617.76 -45102.99
