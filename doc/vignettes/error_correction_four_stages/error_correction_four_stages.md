Error correction models - Elaboration
================

NOTE: this example workflow is outdated and will be revised, for more relevant example modelling workflows go to [this list](https://github.com/jmp75/streamflow-forecasting-tools-onboard/blob/master/doc/sample_workflows.md)


Elaboration 
===========

``` r
library(swift)

areasKm2 <- c(1)
mss <- createTestCatchmentStructure (nodeIds=paste0('node', 1:2), 
  linkIds = paste0('l', 1), fromNode = 'node1', toNode = 'node2', 
  areasKm2 = areasKm2,
  runoffModel='NetRainfall')

mss <- mss$model

setErrorCorrectionModel(mss, 'ERRIS', 'node.node2', length=-1, seed=0)
getVariableIds(mss, 'node.node2.ec')
```

    ##  [1] "Input"          "Observation"    "Lambda"         "Epsilon"       
    ##  [5] "Sigma1_Rising"  "Sigma2_Rising"  "Weight_Rising"  "Sigma1_Falling"
    ##  [9] "Sigma2_Falling" "Weight_Falling" "Mu"             "D"             
    ## [13] "MNoise"         "SNoise"         "Rho"            "Z_s1"          
    ## [17] "Z_s2"           "Z_s3"           "Z_s4"           "Z_o"           
    ## [21] "Z_c"            "CensThr"        "MaxObs"         "Generating"    
    ## [25] "S2"             "Rising"         "Input"          "Updated"       
    ## [29] "Corrected"

``` r
setSimulationTimeStep(mss, 'daily')

errorModelElementId <- 'node.node2';
rainfallVarIdent <- 'subarea.l1.rainfall';
outputVarIdent <- 'node.node2.ec.Updated';

errisModelIdent <- 'node.node2.ec';

errisVarNames <- paste0(errisModelIdent, 
    c(
        '.Lambda'        ,      
        '.Epsilon'       ,      
        '.D'             ,      
        '.Mu'            ,      
        '.Rho'           ,      
        '.Sigma1_Rising' ,
        '.Sigma2_Rising' ,
        '.Weight_Rising' ,
        '.Sigma1_Falling',      
        '.Sigma2_Falling',      
        '.Weight_Falling',
        '.MaxObs'
    )
)

setStateValue(mss, errisVarNames, 
  c( -0.049585152405120937,
  -2.5618878759565269,
  1.2616823384721356,
  -1.3497887589701953,
  0.99999996185640161,
  -1.8648574008829044,
  -2.6261506644992521,
  0.64203563374792605,
  -3.4484072804266788,
  -2.2306669578623604,
  0.62021656472387710,
  38
  )
)

obsRainData <- c(
0.779,
0.7805,
0.7821,
0.795,
0.808,
0.8209,
0.8339,
0.8469,
0.8598,
0.9298)

obsRainData <- obsRainData * 86.4 # TODO revise names perhaps

obsTsData <- c(
3.8339,
3.8318,
3.8255,
3.8143,
3.8039,
3.7943,
3.7788,
3.8162,
3.9882,
4.1236)

warmupStart <- uchronia::mkDate(2000,01,01)
warmupEnd <- uchronia::mkDate(2000,01,10)
forecastStart <- uchronia::mkDate(2000,01,11)
obsRainData <- uchronia::mkDailySeries(warmupStart, obsRainData)
obsTsData <- uchronia::mkDailySeries(warmupStart, obsTsData)

playInput(mss, obsRainData, rainfallVarIdent)

emr <- prepareEnsembleModelRunner(mss, warmupStart, warmupEnd, obsTsData, errorModelElementId)

ensembleSize <- 2;
forecastHorizonLength <- 5;

setupEnsembleModelRunner(emr, forecastStart, ensembleSize, forecastHorizonLength);
recordEnsembleModelRunner(emr, outputVarIdent)

d <- c(0.9997,
        1.0842,
        1.1832,
        1.2788,
        1.3738)

forecastData <- data.frame(d, d+0.1)*86.4
forecastData
```

    ##           d   d...0.1
    ## 1  86.37408  95.01408
    ## 2  93.67488 102.31488
    ## 3 102.22848 110.86848
    ## 4 110.48832 119.12832
    ## 5 118.69632 127.33632

``` r
dummyDate <- warmupStart

# errisParams <-  estimateERRISParameters(
#   simulation = mss, 
#   observedTimeSeries = obsTsData, 
#   errorModelElementId = errorModelElementId, 
#   warmupStart=dummyDate, 
#   warmupEnd=dummyDate, 
#   warmup=FALSE, 
#   estimationStart=warmupStart, 
#   estimationEnd=warmupEnd, 
#   censThr=0, 
#   exclusionStart=dummyDate, 
#   exclusionEnd=dummyDate, 
#   exclusion=FALSE, 
#   terminationCondition=NULL, 
#   hydroParams=NULL)
```
