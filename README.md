<pre>
/***************************************************************************
 *   Copyright (C) by ETHZ/SED                                             *
 *                                                                         *
 *   You can redistribute and/or modify this program under the             *
 *   terms of the "SED Public License for Seiscomp Contributions"          *
 *                                                                         *
 *   You should have received a copy of the "SED Public License for        *
 *   Seiscomp Contributions" with this. If not, you can find it at         *
 *   http://www.seismo.ethz.ch/static/seiscomp_contrib/license.txt         *
 *                                                                         *
 *   This program is distributed in the hope that it will be useful,       *
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of        *
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the         *
 *   "SED Public License for Seiscomp Contributions" for more details      *
 *                                                                         *
 *   Developed by Luca Scarabello, Tobias Diehl                            *
 ***************************************************************************/
</pre>
# Description

The SCRTDD module implements a Real Time Double-Difference Event Location method as described in the paper "Near-Real-Time Double-Difference Event Location Using Long-Term Seismic Archives, with Application to Northern California" by Felix Waldhauser.

This module can also be used to perform (non-real-time) full event catalog Double-Difference relocation, which is covered by another paper: "A Double-Difference Earthquake Location Algorithm: Method and Application to the Northern Hayward Fault, California" by Felix Waldhauser et al.

The actual Double-Difference inversion method is currently performed by the hypoDD software, which has to be installed in the system (currently tested on version 1.3 and 2.1b). We might move away from hypoDD eventually and implement the inversion inside SCRTDD itself. Nevertheless we want to make the transition from hypoDD to SCRTDD software easier and the fact that SCRTDD uses hypoDD internally allows the users to make use of their existing hypoDD experience (configuration, expected behaviour and so on).

## Compile

In order to use this module the sources have to be compiled to an executable. Merge them into the Seiscomp3 sources and compile Seiscomp3 as usual.
<pre>
# merge rtdd-addons and Seiscomp3
git clone https://github.com/SeisComP3/seiscomp3.git sc3-src
cd sc3-src
git submodule add -f https://gitlab.seismo.ethz.ch/lucasca/rtdd-addons.git src/rtdd
</pre>
For compiling Seiscomp3, please refer to https://github.com/SeisComP3/seiscomp3#compiling



# Getting Started

## 1. Install hypoDD software

Go to https://www.ldeo.columbia.edu/~felixw/hypoDD.html and dowload hypoDD. Untar the sources, go to src folder, run make. Done! 

Please remember to set the Hypodd array limits (compilation time options available via include/*inc files) accordingly with the size of your problem.


## 2. Define a background catalog for real time relocations

New origins will be relocated in real time against a background catalog of high quality locations. Those high quality events that form the background catalog can be already present in seiscompo database or not. In the latter case the background catalog has to be generated. Either way, the catalog has to be specified in scrtdd when configuring it.

![Catalog selection option](/img/catalog-selection1.png?raw=true "Catalog selection from event/origin ids")

If we already have a high quality catalog, then we can easily specify it in scrtdd configuration as a path to a file containing a list of origin id or event id (in which case the preferred origin will be used). The file format must be a csv file in which there must be at least one column called seiscompId, from which the ids will be fetched by scrtdd.

E.g. *file myCatalog.csv*

```
seiscompId
event2019ducmfd
Origin/20181214107387.056851.253104
event2019sfamfd
Origin/20180053105627.031726.697885
Origin/20190121103332.075405.6234534
event2019dubnfr
Origin/20190223103327.031726.346363
```

In the more general case in which we don't already have a background catalog, we have to build one using scrtdd. Generating a background catalog involves two steps: first we select the candidate events that might have low quality locations and then we use scrtdd to relocated those to achieve the quality needed for a background catalog. 

Once the candidate events have been selected, we write their seiscomp ids in a file using the same format as the example above (myCatalog.csv). Once that's done, we run the command:

```
scrtdd --dump-catalog myCatalog.csv
```

It is usually convenient to see the logs on the console, for this you can simply add the options ```--verbosity=3 --console=1``` to the command.

Depending on your configuration, the database could be provided via scmaster instead of global.cfg, in this case you have to pass the database connection via command line option. This is also useful if the events resides on a different database machine. Let's use the -d option to specify the database connection:

```
scrtdd --dump-catalog myCatalog.csv  -d  mysql://user:password@host/seiscompDbName
```

The above command will generate three files (event.csv, phase.csv and stations.csv) which contain the information needed by scrtdd to relocate the selected events. 

Note: those file use the extended catalog format, useful when the catalog information is fully contained in those files. Compare to the previous example format (list of event/origin ids) the extended format doesn't require the catalog to be present on the seiscomp database. This might come in handy when using events coming from a different seiscomp installation e.g the events can be dumped (--dump-catalog) and used in another machine. Also those file can be easily generated from another source.

E.g. *file event.csv*

```
id,isotime,latitude,longitude,depth,magnitude,horizontal_err,vertical_err,rms
1,2014-01-10T04:46:47.689331Z,46.262846,7.400132,8.6855,1.63,0.0000,0.0000,0.1815
2,2014-01-19T05:24:26.754208Z,46.264482,7.404143,8.4316,0.94,0.0000,0.0000,0.1740
3,2014-02-21T04:05:27.03289Z,46.266118,7.402066,7.3145,0.37,0.0000,0.0000,0.1177
4,2014-04-02T17:05:28.141739Z,46.262846,7.408248,7.0098,0.42,0.0000,0.0000,0.1319
```

E.g. *file station.csv*

```
id,latitude,longitude,elevation,networkCode,stationCode
4DAG01,46.457412,8.079460,2358.0,4D,AG01
4DAG02,46.460620,8.078122,2375.0,4D,AG02
4DAG03,46.458288,8.075408,2369.0,4D,AG03
4DBSG1,46.107760,7.732020,3378.0,4D,BSG1
```

E.g. *file phase.csv*

```
eventId,stationId,isotime,lowerUncertainty,upperUncertainty,type,networkCode,stationCode,locationCode,channelCode,evalMode
1,CHSIMPL,2014-01-10T04:47:02.000765Z,0.100,0.100,Sg,CH,SIMPL,,HHT,manual
1,CHNALPS,2014-01-10T04:47:06.78218Z,0.100,0.100,P1,CH,NALPS,,HHR,manual
1,CHBNALP,2014-01-10T04:47:05.918759Z,0.200,0.200,P1,CH,BNALP,,HHZ,automatic
1,CHFUSIO,2014-01-10T04:47:04.812236Z,0.100,0.100,Pg,CH,FUSIO,,HHR,manual
1,FRRSL,2014-01-10T04:47:13.089093Z,0.200,0.200,Sg,FR,RSL,00,HHT,manual
1,FRRSL,2014-01-10T04:47:02.689842Z,0.050,0.050,Pg,FR,RSL,00,HHZ,automatic
1,CHGRIMS,2014-01-10T04:47:01.597023Z,0.100,0.100,Pg,CH,GRIMS,,HHR,manual
1,IVMRGE,2014-01-10T04:46:58.219541Z,0.100,0.100,Pg,IV,MRGE,,HHR,manual
```

Now that we have dumped the events (event.csv, phase.csv, stations.csv) we might perform some editing of those files, if required, then we relocate them. To do so we need to create a new profile inside scrtdd configuration and then we have to set the generated files (event.csv, phase.csv, stations.csv) as the catalog of the profile.

![Catalog selection option](/img/catalog-selection2.png?raw=true "Catalog selection from raw file format")

At this point we have to configure the other profile options that control the relocation process. Since we are performing a multi event relocation, we have to only configure the `step2options` (see the next paragraph on real time single event relocation to understand step1 and step2 options).

![Relocation options](/img/multiEventStep2options.png?raw=true "Relocation options")
![Relocation options](/img/xcorr.png?raw=true "Relocation options")  

Once we are happy witht he options, we can relocate the catalog with the command:

```
scrtdd --reloc-profile profileName
```

scrtdd will relocated the catalog and will generate another set of files reloc-event.csv reloc-phase.csv and reloc-stations.csv, which together define a new catalog with relocated origins. At this point we should check the relocated events and see if we are happy with the results. If not, we change scrtdd settings and relocate the catalog again until we are satisfied with the locations, at which point we can finally set the resulting relocated catalog as background catalog in our scrtdd profile (overwriting the previous file triples event.csv, phase.csv, stations.csv that is no longer needed for real time relocation).

We are now ready to perform real time relocation!


Note 1: To help figuring out the right values for cross correlation, waveform filtering and signal to noise ratio options, two command lines options come in handy:


```
scrtdd --help
  --load-profile-wf arg                 Load catalog waveforms from the 
                                        configured recordstream and save them 
                                        into the profile working directory
  --debug-wf                            Enable the saving of waveforms 
                                        (filtered/resampled, SNR rejected, ZRT 
                                        projected and scrtdd detected phase) 
                                        into the profile working directory. 
                                        Useful when run in combination with 
                                        --load-profile-wf
```

One way to take adavantage of the options is to add --debug-wf when relocating the catalog and all the used waveforms will be written to disk as miniseed file for inspection:

```
scrtdd --reloc-profile profileName --debug-wf
```

If you want to inspect all the waveforms contained in the catalog, that is not only the ones used during multi event relocation, it is possible to use the --load-profile-wf option too. This option is also useful to force scrtdd to load all the waveforms and store them on disk so that they will be already available in real time relocation:

```
scrtdd --debug-wf --load-profile-wf profileName
```


Note 2: it is possible to use ph2dt utility to perform the clustering. It this case the scrtdd configuration `step2options.clusteing.*` will not be used. Instead, ph2dt will be run to generate dt.ct file, and for each entry in the generated dt.ct file the cross correlation will be performed and the relative dt.cc file created.

```
scrtdd --reloc-profile profileName --use-ph2dt /some/path/ph2dt.inp [--ph2dt-path /some/path/ph2dt]
```


There are also some other interesting catalong related options:


```
scrtdd --help

Catalog:
  --dump-catalog arg                    Dump the seiscomp event/origin id file 
                                        passed as argument into a catalog file 
                                        triplet (station.csv,event.csv,phase.cs
                                        v)
  --dump-catalog-xml arg                Convert the input catalog into XML 
                                        format. The input can be a single file 
                                        (containing seiscomp event/origin ids) 
                                        or a catalog file triplet 
                                        (station.csv,event.csv,phase.csv)
  --merge-catalogs arg                  Merge in a single catalog all the 
                                        catalog file triplets 
                                        (station1.csv,event1.csv,phase1.csv,sta
                                        tion2.csv,event2.csv,phase2.csv,...) 
                                        passed as arguments

MultiEvents:
  --reloc-profile arg                   Relocate the catalog of profile passed 
                                        as argument
  --ph2dt-path arg                      Specify path to ph2dt executable
  --use-ph2dt arg                       When relocating a catalog use ph2dt. 
                                        This option requires a ph2dt control 
                                        file
  --no-overwrite                        When relocating a profile don't 
                                        overwrite existing files in the working
                                        directory (avoid re-computation and 
                                        allow manual editing)
```

## 3. Real time single origin relocation

Real time relocation uses the same configuration we have seen in full catalog relocation, but real time relocation is done in two steps, each one controlled by a specific configuration.

Step 1: location refinement. In this step scrtdd performs a preliminary relocation of the origin using only catalog absolute travel time entries (dt.ct only).

Step 2: the refined location is used as starting location to perform a more precise relocation using both catalog absolute travel times (dt.ct) and differential travel times from cross correlation (dt.cc). 

After step2 the relocated origin is sent to the messaging system. If step2 fails, then the relocated origin from step1 is sent to the messaging system. If step1 fails, step2 is attempted anyway.

Note: when performing the multi event (catalog) relocation ("scrtdd --reloc-profile") only step2options are considered.


![Relocation options](/img/step1options.png?raw=true "Relocation options")
![Relocation options](/img/step2options.png?raw=true "Relocation options")
![Relocation options](/img/xcorr.png?raw=true "Relocation options")  



To test the real time relocation we can use two command line options which relocate existing origins:

```
scrtdd --help

SingleEvent:
  -O [ --origin-id ] arg                Relocate the origin (or multiple 
                                        comma-separated origins) and send a 
                                        message. Each origin will be processed 
                                        accordingly with the matching profile 
                                        region unless --profile option is used
  --ep arg                              Event parameters XML file for offline 
                                        processing of contained origins (imply 
                                        test option). Each contained origin 
                                        will be processed accordingly with the 
                                        matching profile region unless 
                                        --profile option is used

```

E.g. if we want to process an origin we can run the following command and then check on scolv the relocated origin (the messaging system must be active):


```
scrtdd -O someOriginId
```

Alternatively we can reprocess an XML file:

```
scrtdd --ep eventparameter.xml
```

## 4. RecordStream configuration

SeisComP3 applications access waveform data through the RecordStream interface and it is usually configured in global.cfg, for example:

```
recordstream = combined://slink/localhost:18000;sdsarchive//mnt/miniseed
```

This configuration is a combination of seedlink and sds archive, which is very useful because scrtdd catalog waveforms can be retieved via sds while reat time event data can be fetched via seedlink (much faster since recent data is probably already in memory).

The seedlink service can sometime delay the relocation of incoming origin due to timeouts. For this reason we suggest to pass to configuration option: timeout and retries

e.g. Here we force a timeout of 2 seconds(default is 5 minutes) and do not try to reconnect. 

```
recordstream = combined://slink/localhost:18000?timeout=2&retries=0;sdsarchive//rz_nas/miniseed
```

## 5. Locator plugin

A (re)locator plugin is also avaiable in the code, which makes scrtdd available via scolv. To enable this plugin just add `rtddloc` to the list of plugins in the global configuration.

![Locator plugin](/img/locator-plugin.png?raw=true "Locator plugin")

## 6. Troubleshooting

Check log file: ~/.seiscomp/log/scrtdd.log 

Alternatively, when running scrtdd from the command line use the following options:

```
# set log level to info (3), or debug (4) and log to the console (standard output) insted of log file
--verbosity=3 --console=1
```

A useful option we can find in scrtdd configuration is `keepWorkingFiles`, which prevent the deletion of scrtdd processing files from the working directory. In this way we can access the working folder and check input, output files used for running hypodd. Make sure to check the `*.out` files, which contain the console output of hypodd (sometimes we can find errors only in there, as they do not appear in hypodd.log file).

Another useuful command line option is --dump-wf, which allows to dump the waveforms used for cross correlation after the filtering and resampling have been applied.

Finally, remember to set the Hypodd array limits (compilation time options available via *inc files) accordingly with the size of your problem. If the full catalog relocation doesn't seem to relocate at all, you might have probably hit array limits.
