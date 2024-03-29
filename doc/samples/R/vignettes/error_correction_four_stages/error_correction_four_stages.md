Error correction - ERRIS
================
Jean-Michel Perraud
2020-01-28

# Error correction models - ERRIS

## About this document

This document was generated from an R markdown file on 2020-01-28
10:52:38.

[Li, Ming; Wang, QJ; Bennett, James; Robertson, David. Error reduction
and representation in stages (ERRIS) in hydrological modelling for
ensemble streamflow forecasting. Hydrology and Earth System
Sciences. 2016; 20:3561-3579.
https://doi.org/10.5194/hess-20-3561-2016](https://doi.org/10.5194/hess-20-3561-2016)

## Calibrating ERRIS

### Model structure

We use sample hourly data from the Adelaide catchment [this catchment in
the Northern Territory,
TBC](https://en.wikipedia.org/wiki/Adelaide_River). The catchment model
set up is not the key point of this vignette so we do not comment on
that
section:

``` r
library(swift)
```

``` r
catchmentStructure <- sampleCatchmentModel(siteId = "Adelaide", configId = "catchment")

hydromodel <- "GR4J";
channel_routing <- 'MuskingumNonLinear';
hydroModelRainfallId <-'P'
hydroModelEvapId <-'E'

# set models
insimulation <- swapModel(catchmentStructure, modelId = hydromodel ,what = "runoff")
simulation <- swapModel(insimulation, modelId = channel_routing ,what = "channel_routing")

saId <- getSubareaIds(simulation)
stopifnot(length(saId) == 1)

precipTs <- sampleSeries(siteId = "Adelaide", varName = "rain")
evapTs <- sampleSeries(siteId = "Adelaide", varName = "evap")
flowRateTs <- sampleSeries(siteId = "Adelaide", varName = "flow")

playInput(simulation, precipTs, mkFullDataId('subarea', saId, hydroModelRainfallId))
playInput(simulation, evapTs, mkFullDataId('subarea', saId, hydroModelEvapId))
configureHourlyGr4j(simulation)
setSimulationTimeStep(simulation, 'hourly')

# Small time interval only, to reduce runtimes in this vignette
simstart <- uchronia::mkDate(2010,12,1)  
simend <- uchronia::mkDate(2011,6,30,23)  
simwarmup <- simstart
setSimulationSpan(simulation, simstart, simend)
```

``` r
templateHydroParameterizer <- function(simulation) {
  calibragem::defineParameterizerGr4jMuskingum(refArea=250, timeSpan=3600L, simulation=simulation, paramNameK="Alpha", objfun='NSE', deltaT=1L)
}

nodeId <- 'node.2'
flowId <- mkFullDataId(nodeId, 'OutflowRate')

recordState(simulation, flowId)
```

We use pre-calibrated hydrologic parameters (reproducible with
doc/error\_correction\_doc\_preparation.r in this package structure)

``` r
p <- templateHydroParameterizer(simulation)
setMinParameterValue(p, 'R0', 0.0)
setMaxParameterValue(p, 'R0', 1.0)
setMinParameterValue(p, 'S0', 0.0)
setMaxParameterValue(p, 'S0', 1.0)
SetParameterValue_R( p, 'log_x4', 1.017730e+00)
SetParameterValue_R( p, 'log_x1', 2.071974e+00  )
SetParameterValue_R( p, 'log_x3', 1.797909e+00  )
SetParameterValue_R( p, 'asinh_x2', -1.653842e+00)  
SetParameterValue_R( p, 'R0', 2.201930e-11  )
SetParameterValue_R( p, 'S0', 3.104968e-11  )
SetParameterValue_R( p, 'X', 6.595537e-03   ) # Gotcha: needs to be set before alpha is changed.
SetParameterValue_R( p, 'Alpha', 6.670534e-01   )
parameterizerAsDataFrame(p)
```

    ##       Name         Min        Max         Value
    ## 1   log_x4  0.00000000 2.38021124  1.017730e+00
    ## 2   log_x1  0.00000000 3.77815125  2.071974e+00
    ## 3   log_x3  0.00000000 3.00000000  1.797909e+00
    ## 4 asinh_x2 -3.98932681 3.98932681 -1.653842e+00
    ## 5       R0  0.00000000 1.00000000  2.201930e-11
    ## 6       S0  0.00000000 1.00000000  3.104968e-11
    ## 7        X  0.00100000 0.01662228  6.595537e-03
    ## 8    Alpha  0.01116157 1.68112917  6.670534e-01

``` r
sViz <- uchronia::mkDate(2010,12,1)
eViz <- uchronia::mkDate(2011,4,30,23)

oneWetSeason <- function(tts) {
    window(tts, start=sViz, end=eViz) 
}

obsVsCalc <- function(obs, calc, ylab="flow (m3/s)") {
  obs <- oneWetSeason(obs)
  calc <- oneWetSeason(calc)
    joki::plotTwoSeries(obs, calc, ylab=ylab, startTime = start(calc), endTime = end(calc))
}

applySysConfig(p, simulation)
execSimulation(simulation)
obsVsCalc(flowRateTs, getRecorded(simulation, flowId))
```

<img src="./error_correction_four_stages_files/figure-gfm/unnamed-chunk-6-1.png" style="display:block; margin: auto" style="display: block; margin: auto;" />

### Set up the error correction model

``` r
list(getNodeIds(simulation), 
getNodeNames(simulation))
```

    ## [[1]]
    ## [1] "2" "1"
    ## 
    ## [[2]]
    ## [1] "Outlet" "Node_1"

``` r
errorModelElementId <- 'node.2';
setErrorCorrectionModel(simulation, 'ERRIS', errorModelElementId, length=-1, seed=0)

flowRateTsGapped <- flowRateTs
flowRateTsGapped['2011-02'] <- NA

# plot(flowRateTsGapped)

playInput(simulation,flowRateTsGapped,varIds=mkFullDataId(errorModelElementId,"ec","Observation"))
```

Now, prepare a model with error correction, and set up for generation

``` r
ecs <- swift::CloneModel_R(simulation)

setStateValue(ecs,mkFullDataId(nodeId,"ec","Generating"),FALSE)
updatedFlowVarID <- mkFullDataId(nodeId,"ec","Updated")
inputFlowVarID <- mkFullDataId(nodeId,"ec","Input")
recordState(ecs,varIds=c(updatedFlowVarID, inputFlowVarID))
```

### ERRIS calibration in stages

``` r
#termination <- getMaxRuntimeTermination(0.005)
termination <- swift::CreateSceTerminationWila_Pkg_R('relative standard deviation', c('0.05','0.0167'))
```

We could set up a four-stages estimation in one go, but we will instead
work in each stages for didactic purposes.

``` r
censOpt = 0.0
estimator <- createERRISParameterEstimator (simulation, flowRateTs, errorModelElementId,
                                            estimationStart = simstart, estimationEnd=simend, censThr=0.0, censOpt,
                                            termination, restrictionOn=TRUE, weightedLeastSquare=FALSE)

stageOnePset = CalibrateERRISStageOne_R(estimator)
print(parameterizerAsDataFrame(stageOnePset))
```

    ##              Name    Min    Max       Value
    ## 1         Epsilon  -20.0    0.0   -8.676488
    ## 2          Lambda  -30.0    5.0   -1.451523
    ## 3               D    0.0    0.0    0.000000
    ## 4              Mu    0.0    0.0    0.000000
    ## 5             Rho    0.0    0.0    0.000000
    ## 6  Sigma1_Falling    0.0    0.0    0.000000
    ## 7   Sigma1_Rising    0.0    0.0    0.000000
    ## 8  Sigma2_Falling    0.0    0.0    0.000000
    ## 9   Sigma2_Rising    0.0    0.0    0.000000
    ## 10 Weight_Falling    1.0    1.0    1.000000
    ## 11  Weight_Rising    1.0    1.0    1.000000
    ## 12        CensThr    0.0    0.0    0.000000
    ## 13        CensOpt    0.0    0.0    0.000000
    ## 14         MaxObs 1126.3 1126.3 1126.300000

#### Stage 2

Stage two can be logged:

``` r
SetERRISVerboseCalibration_R(estimator, TRUE)

stageTwoPset = CalibrateERRISStageTwo_R(estimator, stageOnePset)
print(parameterizerAsDataFrame(stageTwoPset))
```

    ##              Name         Min         Max        Value
    ## 1               D    0.000000    2.000000    0.7386187
    ## 2              Mu -100.000000  100.000000   -3.5225392
    ## 3   Sigma1_Rising   -6.907755    6.907755    0.9109061
    ## 4         CensOpt    0.000000    0.000000    0.0000000
    ## 5         CensThr    0.000000    0.000000    0.0000000
    ## 6         Epsilon   -8.676488   -8.676488   -8.6764883
    ## 7          Lambda   -1.451523   -1.451523   -1.4515233
    ## 8          MaxObs 1126.300000 1126.300000 1126.3000000
    ## 9             Rho    0.000000    0.000000    0.0000000
    ## 10 Sigma1_Falling    0.000000    0.000000    0.0000000
    ## 11 Sigma2_Falling    0.000000    0.000000    0.0000000
    ## 12  Sigma2_Rising    0.000000    0.000000    0.0000000
    ## 13 Weight_Falling    1.000000    1.000000    1.0000000
    ## 14  Weight_Rising    1.000000    1.000000    1.0000000

``` r
mkEcIds <- function(p) {
  df <- parameterizerAsDataFrame(p)
  df$Name <- mkFullDataId(nodeId, 'ec', df$Name)
  createParameterizer('Generic',df)
}

applySysConfig(mkEcIds(stageTwoPset), ecs)
execSimulation(ecs)
obsVsCalc(flowRateTsGapped, getRecorded(ecs, updatedFlowVarID))
```

<img src="./error_correction_four_stages_files/figure-gfm/unnamed-chunk-13-1.png" style="display:block; margin: auto" style="display: block; margin: auto;" />

A helper function to process the calibration log:

``` r
prepOptimLog <- function(estimator, fitnessName = "Log.likelihood")
{
  optimLog = getLoggerContent(estimator)
  # head(optimLog)
  optimLog$PointNumber = 1:nrow(optimLog)   
  logMh <- mhplot::mkOptimLog(optimLog, fitness = fitnessName, messages = "Message", categories = "Category") 
  geomOps <- mhplot::subsetByMessage(logMh)
  d <- list(data=logMh, geomOps=geomOps)
}
```

``` r
d <- prepOptimLog(estimator, fitnessName = "Log.likelihood")
print(mhplot::plotParamEvolution(d$geomOps, 'Sigma1_Rising', c(0, max(d$data@data$Log.likelihood))))
```

<img src="./error_correction_four_stages_files/figure-gfm/unnamed-chunk-15-1.png" style="display:block; margin: auto" style="display: block; margin: auto;" />

#### Stage 3

``` r
stageThreePset = CalibrateERRISStageThree_R(estimator, stageTwoPset)
print(parameterizerAsDataFrame(stageThreePset))
```

    ##              Name          Min          Max        Value
    ## 1             Rho    0.0000000    1.0000000    0.9880881
    ## 2   Sigma1_Rising   -6.9077553    6.9077553   -0.9134793
    ## 3         CensOpt    0.0000000    0.0000000    0.0000000
    ## 4         CensThr    0.0000000    0.0000000    0.0000000
    ## 5               D    0.7386187    0.7386187    0.7386187
    ## 6         Epsilon   -8.6764883   -8.6764883   -8.6764883
    ## 7          Lambda   -1.4515233   -1.4515233   -1.4515233
    ## 8          MaxObs 1126.3000000 1126.3000000 1126.3000000
    ## 9              Mu   -3.5225392   -3.5225392   -3.5225392
    ## 10 Sigma1_Falling    0.0000000    0.0000000    0.0000000
    ## 11 Sigma2_Falling    0.0000000    0.0000000    0.0000000
    ## 12  Sigma2_Rising    0.0000000    0.0000000    0.0000000
    ## 13 Weight_Falling    1.0000000    1.0000000    1.0000000
    ## 14  Weight_Rising    1.0000000    1.0000000    1.0000000

``` r
d <- prepOptimLog(estimator, fitnessName = "Log.likelihood")
print(mhplot::plotParamEvolution(d$geomOps, 'Rho', c(0, max(d$data@data$Log.likelihood))))
```

<img src="./error_correction_four_stages_files/figure-gfm/unnamed-chunk-17-1.png" style="display:block; margin: auto" style="display: block; margin: auto;" />

#### Stage 3a, generating and fitting M and S if free

``` r
stageThreePsetMS = CalibrateERRISStageThreeMS_R(estimator, stageThreePset)
print(parameterizerAsDataFrame(stageThreePsetMS))
```

    ##              Name          Min          Max        Value
    ## 1             Rho    0.0000000    1.0000000    0.9880881
    ## 2   Sigma1_Rising   -6.9077553    6.9077553   -0.9134793
    ## 3         CensOpt    0.0000000    0.0000000    0.0000000
    ## 4         CensThr    0.0000000    0.0000000    0.0000000
    ## 5               D    0.7386187    0.7386187    0.7386187
    ## 6         Epsilon   -8.6764883   -8.6764883   -8.6764883
    ## 7          Lambda   -1.4515233   -1.4515233   -1.4515233
    ## 8          MaxObs 1126.3000000 1126.3000000 1126.3000000
    ## 9              Mu   -3.5225392   -3.5225392   -3.5225392
    ## 10 Sigma1_Falling    0.0000000    0.0000000    0.0000000
    ## 11 Sigma2_Falling    0.0000000    0.0000000    0.0000000
    ## 12  Sigma2_Rising    0.0000000    0.0000000    0.0000000
    ## 13 Weight_Falling    1.0000000    1.0000000    1.0000000
    ## 14  Weight_Rising    1.0000000    1.0000000    1.0000000
    ## 15         MNoise -100.0000000  100.0000000   -2.5833943
    ## 16         SNoise  -10.0000000   10.0000000    1.8858371

``` r
applySysConfig(mkEcIds(stageThreePsetMS), ecs)
execSimulation(ecs)
obsVsCalc(flowRateTsGapped, getRecorded(ecs, updatedFlowVarID))
```

<img src="./error_correction_four_stages_files/figure-gfm/unnamed-chunk-19-1.png" style="display:block; margin: auto" style="display: block; margin: auto;" />

#### Stage 4, rising limb

``` r
stageFourPsetRising = CalibrateERRISStageFour_R(estimator, stageThreePsetMS, useRising = TRUE)
print(parameterizerAsDataFrame(stageFourPsetRising))
```

    ##              Name          Min          Max        Value
    ## 1   Sigma1_Rising   -6.9077553    6.9077553   -1.0822514
    ## 2   Sigma2_Rising   -6.9077553    6.9077553    0.6662236
    ## 3   Weight_Rising    0.5000000    1.0000000    0.8770708
    ## 4         CensOpt    0.0000000    0.0000000    0.0000000
    ## 5         CensThr    0.0000000    0.0000000    0.0000000
    ## 6               D    0.7386187    0.7386187    0.7386187
    ## 7         Epsilon   -8.6764883   -8.6764883   -8.6764883
    ## 8          Lambda   -1.4515233   -1.4515233   -1.4515233
    ## 9          MaxObs 1126.3000000 1126.3000000 1126.3000000
    ## 10             Mu   -3.5225392   -3.5225392   -3.5225392
    ## 11            Rho    0.9880881    0.9880881    0.9880881
    ## 12 Sigma1_Falling    0.0000000    0.0000000    0.0000000
    ## 13 Sigma2_Falling    0.0000000    0.0000000    0.0000000
    ## 14 Weight_Falling    1.0000000    1.0000000    1.0000000

``` r
d <- prepOptimLog(estimator, fitnessName = "Log.likelihood")
print(mhplot::plotParamEvolution(d$geomOps, 'Weight_Rising', c(0, max(d$data@data$Log.likelihood))))
```

<img src="./error_correction_four_stages_files/figure-gfm/unnamed-chunk-21-1.png" style="display:block; margin: auto" style="display: block; margin: auto;" />

``` r
applySysConfig(mkEcIds(stageFourPsetRising), ecs)
execSimulation(ecs)
obsVsCalc(flowRateTsGapped, getRecorded(ecs, updatedFlowVarID))
```

<img src="./error_correction_four_stages_files/figure-gfm/unnamed-chunk-22-1.png" style="display:block; margin: auto" style="display: block; margin: auto;" />

#### Stage 4, falling limbs

``` r
stageFourPsetFalling = CalibrateERRISStageFour_R(estimator, stageThreePsetMS, useRising = FALSE)
print(parameterizerAsDataFrame(stageFourPsetFalling))
```

    ##              Name          Min          Max        Value
    ## 1   Sigma1_Rising   -6.9077553    6.9077553   -3.1021521
    ## 2   Sigma2_Rising   -6.9077553    6.9077553   -0.6290198
    ## 3   Weight_Rising    0.5000000    1.0000000    0.8131939
    ## 4         CensOpt    0.0000000    0.0000000    0.0000000
    ## 5         CensThr    0.0000000    0.0000000    0.0000000
    ## 6               D    0.7386187    0.7386187    0.7386187
    ## 7         Epsilon   -8.6764883   -8.6764883   -8.6764883
    ## 8          Lambda   -1.4515233   -1.4515233   -1.4515233
    ## 9          MaxObs 1126.3000000 1126.3000000 1126.3000000
    ## 10             Mu   -3.5225392   -3.5225392   -3.5225392
    ## 11            Rho    0.9880881    0.9880881    0.9880881
    ## 12 Sigma1_Falling    0.0000000    0.0000000    0.0000000
    ## 13 Sigma2_Falling    0.0000000    0.0000000    0.0000000
    ## 14 Weight_Falling    1.0000000    1.0000000    1.0000000

``` r
Nd <- prepOptimLog(estimator, fitnessName = "Log.likelihood")
print(mhplot::plotParamEvolution(d$geomOps, 'Weight_Rising', c(0, max(d$data@data$Log.likelihood))))
```

<img src="./error_correction_four_stages_files/figure-gfm/unnamed-chunk-24-1.png" style="display:block; margin: auto" style="display: block; margin: auto;" />

#### Final consolidated parameter set

``` r
finalPset = ConcatenateERRISStagesParameters_R(estimator, hydroParams = createParameterizer(), stage1_result =  stageOnePset, stage2_result = stageTwoPset, 
                                   stage3_result = stageThreePsetMS, stage4a_result = stageFourPsetRising, stage4b_result = stageFourPsetFalling, toLongParameterName = FALSE)

print(parameterizerAsDataFrame(finalPset))
```

    ##              Name          Min          Max        Value
    ## 1         CensThr    0.0000000    0.0000000    0.0000000
    ## 2         CensOpt    0.0000000    0.0000000    0.0000000
    ## 3          MNoise -100.0000000  100.0000000   -2.5833943
    ## 4          SNoise  -10.0000000   10.0000000    1.8858371
    ## 5          Lambda   -1.4515233   -1.4515233   -1.4515233
    ## 6         Epsilon   -8.6764883   -8.6764883   -8.6764883
    ## 7              Mu   -3.5225392   -3.5225392   -3.5225392
    ## 8               D    0.7386187    0.7386187    0.7386187
    ## 9             Rho    0.9880881    0.9880881    0.9880881
    ## 10         MaxObs 1126.3000000 1126.3000000 1126.3000000
    ## 11  Sigma1_Rising   -6.9077553    6.9077553   -1.0822514
    ## 12  Sigma2_Rising   -6.9077553    6.9077553    0.6662236
    ## 13  Weight_Rising    0.5000000    1.0000000    0.8770708
    ## 14 Sigma1_Falling   -6.9077553    6.9077553   -3.1021521
    ## 15 Sigma2_Falling   -6.9077553    6.9077553   -0.6290198
    ## 16 Weight_Falling    0.5000000    1.0000000    0.8131939

### Legacy call

Check that the previous “one stop shop” call gives the same results.

``` r
dummyDate <- simstart

psetFullEstimate <- estimateERRISParameters(simulation, flowRateTs, errorModelElementId,
  warmupStart=dummyDate, warmupEnd=dummyDate, warmup=FALSE, estimationStart = simstart, estimationEnd=simend, censThr=0.0,
 censOpt = censOpt, exclusionStart=dummyDate, exclusionEnd=dummyDate, exclusion=FALSE, terminationCondition = termination,
  hydroParams = NULL, errisParams = NULL, restrictionOn = TRUE,
  weightedLeastSquare = FALSE)

print(parameterizerAsDataFrame(psetFullEstimate))
```

    ##                        Name          Min          Max        Value
    ## 1         node.2.ec.CensThr    0.0000000    0.0000000    0.0000000
    ## 2         node.2.ec.CensOpt    0.0000000    0.0000000    0.0000000
    ## 3          node.2.ec.MNoise -100.0000000  100.0000000   -2.5833943
    ## 4          node.2.ec.SNoise  -10.0000000   10.0000000    1.8858371
    ## 5          node.2.ec.Lambda   -1.4515233   -1.4515233   -1.4515233
    ## 6         node.2.ec.Epsilon   -8.6764883   -8.6764883   -8.6764883
    ## 7              node.2.ec.Mu   -3.5225392   -3.5225392   -3.5225392
    ## 8               node.2.ec.D    0.7386187    0.7386187    0.7386187
    ## 9             node.2.ec.Rho    0.9880881    0.9880881    0.9880881
    ## 10         node.2.ec.MaxObs 1126.3000000 1126.3000000 1126.3000000
    ## 11  node.2.ec.Sigma1_Rising   -6.9077553    6.9077553   -1.0822514
    ## 12  node.2.ec.Sigma2_Rising   -6.9077553    6.9077553    0.6662236
    ## 13  node.2.ec.Weight_Rising    0.5000000    1.0000000    0.8770708
    ## 14 node.2.ec.Sigma1_Falling   -6.9077553    6.9077553   -3.1021521
    ## 15 node.2.ec.Sigma2_Falling   -6.9077553    6.9077553   -0.6290198
    ## 16 node.2.ec.Weight_Falling    0.5000000    1.0000000    0.8131939
