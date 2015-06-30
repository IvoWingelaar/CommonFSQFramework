This file describes process of new skim creation. Most importantly it explains how to attach 
services of the CFF to the existing python configuration.

For jet based analyses (ie the jet skim) we want to use basic PAT
configuration, since it merges all the scaterred information into a single
object (pat::Jet). It also takes care of applying JEC.

Note: for different analysis (e.g. track based) we could start with a 
basic configuration file with only source/geometry/gt/... definitions. There
is nothing that forces us to use PAT if we dont need it.

1. Create python config

 Configuration with different jet collections processed by PAT is avaliable in

PhysicsTools/PatAlgos/test/patTuple_addJets_cfg.py

 This file was copied and saved as S0_makePAT_74.py
(CommonFSQFramework/Core/test/SkimConfigurations/Jets directory)

2. Add basic CFF services

at the end of the file append
------
import CommonFSQFramework.Core.customizePAT
process = CommonFSQFramework.Core.customizePAT.customize(process)
------
note: it should also work on regular, ie non-PAT, configurations. If not -
contact CFF developers

Above customization peforms the following
- Add CFF event counters
- Add TFileService
- Remove the edm output module if present in configuration 
(may need fixing, if it doesnt work contact CFF  developers)

Since event data is saved in our trees it makes no sense to produce a standard
edm output (we dont use it in the framework). Keeping the edm output in
configuration will make your crab jobs to run significantly longer (note:
removing the output most likely will prevent large number of CMSSW modules
from running thanks to magic of unscheduled execution)


3. Add tree producers

A tree producer in CFF is a edm plugin calling several miniViews (see src
directory) in order to fill a tree. Since April 2015 there is no need to
compose a tree producer from miniviews by hand. For most uses a generic tree
producer named CFFTreeProducer should be enough. In configuration you need to
provide from 0 to several psets with miniviews configuration. Each pset must
contain miniView type and branch prefix (branches produced by this producer
will start from this string). 


process.JetTree = cms.EDAnalyzer("CFFTreeProducer",
    JetViewPFAK4CHS  = cms.PSet(
        miniView = cms.string("JetView"),
        branchPrefix = cms.untracked.string("PFAK4CHS"),
        (...) # here follows rest of configuration, e.g. inputtags
    ),
    JetViewCaloAK  = cms.PSet(
        miniView = cms.string("JetView"),
        branchPrefix = cms.untracked.string("CaloAK4"),
        (...)
    ),

    (...) # other miniviews
)

See CFFTreeProducer source code (plugins directory) to learn what
miniviews are supported. Please contact us if you want your miniview to be
included

Last thing to do is to make sure our tree producer will be executed. This is
done by another customization function:

process = CommonFSQFramework.Core.customizePAT.addTreeProducer(process, process.JetTree)

3b) Preffered way to configure tree producer

A different way to configure the CFFTreeProducer is to first create it without
any miniviews:

--------------
process.JetTree = cms.EDAnalyzer("CFFTreeProducer")
--------------

and than to fetch and "attach" to it miniViews configuration from python
directory:

----------------
import CommonFSQFramework.Core.JetViewsConfigs
process.JetTree._Parameterizable__setParameters(CommonFSQFramework.Core.JetViewsConfigs.get(["JetViewPFAK4CHS"]))
----------------
 
The "get" method expects a list of configuration names. This way a centralized 
miniviews configuration is avaliable to all skim configurations.

4. Global tag modifications 

  runCrabJobs.py scripts are able to communicate with the python configuration
via shell environment variables. Variable TMFSampleName contains a name of a
sample for which crab jobs are created. Following customization call

process = CommonFSQFramework.Core.customizePAT.customizeGT(process)

tries to read TMFSampleName variable from environment and set the global
tag to the one from samplesDictionary[TMFSampleName]["GT"] variable


5. data/MC customization

 Sometims it is necessary to apply some specific configuration changes when
running on data/MC. This again can be done using "TMFSampleName" env variable

----------------------------
import os
if "TMFSampleName" not in os.environ:
    print "TMFSampleName not found, assuming we are running on MC"
else:
    s = os.environ["TMFSampleName"]
    sampleList=CommonFSQFramework.Core.Util.getAnaDefinition("sam")
    isData =  sampleList[s]["isData"]
    if isData:
        print "Disabling MC-specific features for sample",s
        runOnData(process)    
        removeMCMatching(process, ['All'])  
-----------------











