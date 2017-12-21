:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. note::

   **This technote is not yet published.**

   Report on the delivery, installation, and initial use of an LSST Camera data acquisition (DAQ) test stand at NCSA, July 18-20, 2017.  Includes notes from a discussion of future plans for DAQ work that was held following the installation.

Software Basics
===============

How to integrate with build system
- Delivered as tree of .h and .so files
Releases can be installed on the system by Mike/Gregg; NCSA can choose which is active with a symlink

gcc v4.4.7
- They can rebuild against another gcc version

Jim's OCS bridge code uses C++ and Makefiles

[ ] eups for ctrl_iip?
- JimP wants to run on bare metal
- [ ] How to deploy?
Resubscribing to a different part of the image involves destroying an object and constructing a new one
MikeH believes that 300MB/sec (2 rafts) is the max that can go to one (today) host, but 1 raft is about right

Only one fiber output, so host machine must be shared with MikeH for management (and storage)
Name recorded in shelf manager is used as image metadata to distinguish between DAQ systems

Three types of RTMs:
- FPA (connects to camera with 8 fibers)
- Storage (with up to 24 SSD)
- Hybrid (what we have, has some of both, up to 1 fiber and 6 SSD)
Our Hybrid RTM doesn't have SSD; in V3 of software, we can populate it
Full system will have FPA and Storage and not Hybrid

Transceiver to OTM loops back in breakout box to another RCE in the RTM that emulates the REBs

Distributed Service Interface (zero-configuration networking tool) underlies internal communication, used to transmit to client, underlies API

Addressing of RCEs based on physical/ATCA location: slot/bay/element
- slot 1-16
- bay 0-3 (bay 4 is DTM for switch control)
- element 0 or 2 (not 1)

DAQ Management
==============

Hardware can run multiple versions (partitions) at the same time
- Two partitions of one raft each or one partition of two rafts
- Can reboot each partition separately, each has its own sequencing and storage repository
- But probably don't need that right now

Changing preferences requires a reboot of RCEs
- Generally try to contact DAQ team before rebooting for any other reason
"Site" = all hosts attached to DAQ
- For now, management server
- MikeH feels that all clients should access client libraries from shared filesystem on Summit
Release = all ATCA code, management code, client libraries
- rpt-sdk is internal SDK
- daq-sdk is external SDK
  - current (our choice) and production ("best guess at what should be running") links
  - each release version (R{release number}-V{version number}.{point release}; R releases are generally not compatible)
Releases could be pushed or pulled
RCEs mount NFS from management server or from their own SSD (which contains "fallback release")
"repository" = storage for N-day store
Each client host needs to have default network buffering limit extended (to 8 MB from 256 kB)
Preferences can be set per-RCE or collectively across entire partition
"Interface" defines which REB source (science REB, guiding REB, wavefront REB) is managed by an RCE
Preferences might carry over when an RCE is reassigned to a different partition
"Role" can be readout, emulation, storage, guiding
- Each readout must be matched with an emulation
- We won't use guiding role at NCSA or for any DM
Wavefront sensors are one CCD, but which half is segments 00-07 and which is segments 10-17 is not clear
Sensor 00, 01, 02 go to REB slot 0 etc.
REB sources (SCI) labeled as {bay}/{board slot}
Each RCE can be connected to up to three REB sources
- Interfaces assigned are some combination of 0, 1, 2 as character string in second argument
- SCI locations given as third through fifth arguments

DAQ API
=======

TonyJ CCS manages DAQ; we have to simulate it if we don't have a CCS

C++11
Can throw exceptions, but only in constructors (for now)
Distribution is position-independent, no need for LD_LIBRARY_PATH
Bugs and feature requests go to JIRA, but not sure which
Release notes on documentation Web site and not in distribution (for now)
Software is just for LSST; "rpt" is multi-project

``include``, ``examples``, ``x86/bin``, ``x86/lib``
Will mostly work with ``ims`` library
``daq/Location.hh``
- REB Source: raft bay/REB board
- C strings, not C++ strings
``daq/LocationSet.hh``
- Can handle sets of Sources
- Predefined Science, Wavefront, Guiding sets
Readout is Acquisition + Commit
- Acquisition is movement from REB Source to CDS RCEs
- Commit is writing into storage system
- Overlap in time, nearly synchronized
- Always read from storage, either during or after
Readout triggered from any host; on the Summit the CCS does this
Data committed as Buckets
- Image (root, contains only metadata of summary and points to list of Source buckets)
- Source (contains readout summary of a particular SCI and points to list of Slice buckets)
- Slice (leaf, time-ordered pixel data, length, encode/decode engines)
``ims/Image.hh``, ``ims/Source.hh``, ``ims/Slice.hh``
All buckets have a unique 64-bit identifier (either binary or hex string); images have names from readout trigger which must be unique
Catalog maintains relationship between image names and bucket ids
Code accessing an image needs a reference to a ``Store`` object
- ``Store`` objects are stateless, fast to construct

Number of pixels is variable from image to image and even Source to Source; DAQ sets a maximum of 24 GB
Typically 32 slices per Source
Can always get an upper limit from metadata, even before pixels arrive
Engineering and diagnostic modes may use strange sequencer programs
Emulation (in NCSA test stand) gives a fixed amount of data per Source
Image names guaranteed by DAQ unique in N-day store, but not over all time

``IMS::Images`` class is an iterator over images using ``id()`` to get next
- Takes a snapshot at the instant of construction
- Don't use returned id (which is a bucket) -- get the name from the image
- Also contains reference to the ``Store``
- Images not necessarily returned in time order, need to look in metadata for image time
Source bucket metadata includes:
- Board serial number
- Various registers including pass-through registers
- Definitions are by CCS, need to be passed out of band
- Platform name
- CCDs per Source

Emulator can simulate registers

.. warning:

  Sensors from different vendors have different readout directions for half the chip

Removing an image is an operation on the image (with the store as an argument)

Slices are a (variable length) vector of Stripes
Each stripe contains 16 elements, one per amplifier
Each element contains 18 bits of pixel data, sign-extended to 32 bits
Ordering of stripes in a slice is always in time order
Rows and columns depends on sensor vendor and sequencer program
Slices from Science, Guiding, Wavefront Sources are different classes
- Decode at a CCD level into provided vector of stripes, need to use Slice length from ``slice.stripes()`` to allocate
``IMS::System(partition name).trigger(image name)``
- Returns error code

Asynchronous input:
- Block until input appears
- Process input in callback, returning a decision on abort
- Repeat if no abort
- Process abort, returning a decision on exit
- Repeat unless want to exit
``IMS::Subscriber``
- Construct with store, desired ``LocationSet``, group (for identification for external interrupts)
  - Activates subscription to images at this time
- Call ``wait()`` with buffer to be filled with abort reason when all slices have been committed
  - Normal exit: reason is image name
- System calls back ``process(image)``, ``process(source)``, ``process(slice)``
- Will always return next image, even if delay between ``wait()`` s
- Can look at pending images and flush them from the queue

To make emulated images, need to create buckets using ``encode()`` method

Can generate external interrupts to subscribers using ``IMS::Publisher``
- Specify group identification
- Give a reason

No way to report damage to a slice; slice would not be committed
``wait()`` would not unblock


Group Naming
------------

- DM-
  - Archiver (distinguished by platform and partition)
  - Forwarder
- CA-
  - Diagnostic cluster (maybe 2)
- TS-
  - Wavefront
- OC-
  - Visualization

API for reusable image construction
-----------------------------------

Construct object given row/column metadata
Accept slice for addition to image
Return image when done

Could be external program that takes an image name and writes a file


Early Integration and evolution of test stand
=============================================

- √ 2015-07: Basic OCS communications
- √ 2017-04: Engineering and Facilities Database from Telemetry
- √ 2017-04: Start/end of night
- 2017-07: DAQ to DM using test stand
- 2017-08: Header service
- 2017-08: Control interfaces
- Combine header service with pixels 2017-10 delayed to 2017-12?
- 2017-12: Full image acquisition test delayed to 2018-02
- Insert new #6 (mini-night) on test stand, maybe in 2018-04
- 2018-02 (now later): Spectrograph operational rehearsal
- 2018-04: EFD transformation

Will populate current RTM with SSDs in 6 months
Could send new RTM to replace current one
Then can do playlists of up to roughly 2 weeks

Header service (just technology with incomplete headers) needs a CCS

5b needs OCS event injector

Building 14 slot crate for us means building for everyone
Can wait (?), only needed for full-scale tests
Architecture can be tested earlier by going sub-raft scale

Tony: How do we switch to TAI?
- Linux kernels as of RHEL7 are supposed to allow ``clock_gettime(CLOCK_TAI)``
- May need to set the UTC offset on boot using ``adjtimex()``
- But ``ntpd``/``ptpd`` may take care of this
Test procedures: Confluence, then MagicDraw/JIRA

May need to put DWDM at NCSA, could maybe get 100km spool of fiber

DAQ Registers and structural metadata
=====================================

Current register allocation for test stand:
- READ_ROWS
- READ_COLS
- PRE_ROWS
- PRE_COLS
- POST_ROWS
- POST_COLS
- READ_COLS2
- OVER_ROWS
- OVER_COLS
Not yet one that says whether E2V or ITL, but should be added
Pre/post/read_cols2 are for ROIs
Sequencer programs must follow this convention; not all may do so right now
- Have a flag in a register that says that the sequencer follows conventions and is rectangular
Registers are per-Source (REB)

In some cases, READ_ROWS and READ_COLS are the only things recorded

SegRows, SegCols, SerCols are hard-coded dependent on the type of CCD, not in registers

Segment order is always 10-17, 07-00 in each stripe, although not in API yet

Fallback if we can't figure out that it's rectangular is 1xN

Chris Stubbs is the one defining most of the sequencer programs; need to make sure he and PST understand how data is going to be recorded

If there's some agreement with Tony on register usage, Mike can build into API as methods

When Forwarders retry, do they re-pull from DAQ or re-send from local memory/storage?
- Fetch from DAQ
- Won't be doubling the simultaneous requests
Does DMCS know if data has been committed?
- Not now, but DBB could inform it
Header Service is at the Summit
- Publishes an event with the headers
- Does not include anything from DAQ, only from DDS
- Headers retrieved from EFD during catchup
Unclear if DDS implementation will do message queuing
- GPDF says it is just not configured now but will be
- [ ] Need to clarify when this will happen if it will happen
Deletion could occur when DBB has replicated
- DM does not call deletion API
- CCS listens to DM telemetry, then calls deletion API
- DM could reissue "safely archived" events for old images
- "Safely archived" = replicated within Data Backbone

Future of DAQ
=============

2017-07 installed v2.0
Will start work on N-day store package next week
Possibly sometime before 2018-01: First implementation will use RAM in CDS, mostly for Tony
- Not persistent, not deep
- Allow other machines besides management host to pull
- Folders will show up here
- V3 (preview)
2018-01 install SSD on hybrid RTM
- Emulator should be "prototype complete"
- Create arbitrary images, load them, and create playlists
- Real REB board, CCD emulator could be installed at NCSA
More COBs (14 slot crate) -- and new ones that enable the 8 ports on the front
- Design should be done in 2018-01 with two working prototypes
- Could start production then, replacing test stand COBs and adding others
- 3 14-slot systems: SLAC, NCSA, one to go to Chile
- 2 month production process
- RTM manufacture is not an issue
- Has a system with old COBs for AuxTel and ComCam already
2018-09 delivery date for 14-slot crate

Can hook up another machine today on the 1 Gbit port, make that the administration machine

Jim is requesting hardware for downstream tests; two machines are not firmly allocated; one could be used for CCS components that do not sit on admin server

[ ] Need to put in an LCR to fix milestones for DAQ transfer

Mike might want to use our L1 Complete Test Stand to run his own tests


Jim Parsons Notes
=================

Mike and Gregg installed the DAQ and connected the control node for it on the morning of Tuesday, the 18th of July.
After installation, Greg worked on getting the software side of the system configured that afternoon.

Over the next few days, Gregg and I spent time together exercising the DAQ API by running sample code, Mike gave an important presentation on the DAQ hardware and the API, the Pathfinder schedule was updated, and the Level one system code - the code that interfaces with the DAQ - was reviewed for everyones benefit.
Here are a few specific things that were learned from an NCSA point of view:

- When the image data is pulled from the DAQ, Mike will have it formatted to 32-bit values with the lower half of the data word being 16 bits of pixel magnitude measurement, and then two bits (17 and 18) for flag values.
  (KTL believes that this is 18 bits of pixel value, sign-extended to handle crosstalk-corrected data.)
  The rest of the integer-sized data word will be zfill’ed to zero.
  It was uncertain before last week whether NCSA would be doing the conversion or the SLAC camera group.
  It is nice to have some resolution on this.
- The addressing and reassembly of the individual amplifier output into properly formatted CCDs was studied.
- The overscan pattern(s) of image rows were considered, and it was revealed that overscan pattern is preferably adjustable; that is, not a constant as the Level one software lead expected before the gathering.
  This information was important in that the fetch software does not necessarily provide the means to include this capability right now, but that the software should not implement it but designed to allow for it to be included down the road if desired, when implementation specifics are available.
- CCS is providing accessible registers in the DAQ for passing run-time specific config values to DM so that these values can be included when the image is built into its final format before shipping.
  Overscan pattern is one element in which the registers will be used for passing necessary JIT data to DM.
  (KTL notes that while the fetch software does not need to distinguish between overscan or pre-scan and science pixels, the total number of pixels present and the register metadata needs to be known and passed downstream.)
- The DAQ system code is designed so that fetch can begin before the image is completely read out - this is a time saving measure that may be exploited if it becomes necessary.
- Mike's opinion is that header data for the final format of image files be rendezvoused with image data at NCSA within the distributor components.
  DM has reservations about this approach for now.
- One unexpected comment that Mike made, is his concern about re-fetching an image portion from the DAQ if something were to go wrong in distribution or processing.
  DM expects this to be an extremely rare event.

The week was VERY successful - many questions were answered that had been waiting for answers for some time.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
