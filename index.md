# Common Metadata Problems Found in Raw LSSTCam Files

```{abstract}
There are times where an incorrect header is written to a raw data file. This document explains the different types of metadata errors we have encountered so far,
```

## Introduction

The FITS headers written to the raw data files by the camera acquisition system are defined in {cite:t}`CTN-004`.
Sometimes a header is not set correctly through either the separate header service not being contactable or a Commandable SAL Component (CSC) {cite:p}`LSE-209` not publishing the correct value.
When this happens the `obs_lsst` package must be updated to record the correct header value and allow the header to be corrected such that a retrieval of the raw file from the Butler {cite:p}`2022SPIE12189E..11J` has the correct version and also when the file is ingested into a Butler the correct translated dimension record metadata is recorded.

In this document we will explain the different header problems that have been encountered and their importance.

## Common Header Issues

### Header Service

The most extreme header problem comes from the camera acquisition system not being able to retrieve the header from the Header Service {cite:p}`DMTN-058`.
What generally happens is that some of the files are written with a full header and others are written with all the non-camera headers set to defaults.
This means that the Butler is unable to ingest all the raw files because the exposure metadata is calculated inconsistently.

The only way to fix this is to dump the good header (filtering out the detector specific headers) into a YAML correction file for `obs_lsst`.
There is no tooling to make this a simple one line patch that can be applied semi-automatically.
It is, though, possible to consider a scenario where a special command-line tool could run to generate the corrections file, commit it, and make the pull request (and even run Jenkins).

### Filter Changes

Early on, we encountered many cases where a filter change would happen and the camera would fail to register the state change event.
This would lead to the previous filter being reported in the header.
We made explicit header corrections in `obs_lsst` for these observing day / sequence number combinations to correct the header.

The camera system has been modified such that it no longer relies on a single state change event and we should not get any more incorrect filter headers from the end of 2026.

### Incorrect User-Supplied Metadata

We have had occasions where an observer has specified the incorrect block number for some observations.
Now that most observing is triggered by the scheduler it should be much harder for this to happen but it is always possible for a user to send some incorrect value if they are not using the scheduler.

### CSC Issues

#### Can See Sky

The header with the most far-ranging effect on the observatory is the `VIGN_MIN` header that reports whether the mirror can see the sky or not.
This is a critical header in that, in conjunction with the camera shutter state, it controls whether the data are embargoed or not.
In some cases the CSC calculating the vignetting can return an "unknown" state that results in the data being embargoed even if the files were not looking at sky.
This also affects the butler metadata tables since users would like to be able to query for on sky observations.

It is critical that this header is fixed quickly once discovered.

#### RA/Dec Tracking Headers

In some cases we have found that it is possible for the `RASTART`/`DECSTART`/`TRACKSYS` headers to retain their previous values when a new observation starts.
This can lead to reported `RA`/`DEC` headers showing the actual telescope position that are wildly different from the (incorrectly recorded) requested position ({jira}`SSW-426`).

## Prioritization

The urgency with which a header should be fixed critically depends on the observing mode and whether the incorrect header affects Butler searches and data processing.

Additionally, `OBJECT` and `CWFS` observation types are treated as the most critical observations because these form part of the LSST itself and we are required to maximize our observing efficiency.
Dropping an observation because we need to fix some headers is not acceptable.

The following observation types are treated as best efforts when it comes to fixing bad headers.

* ACQ
* ENGTEST
* FLAT
* SFLAT
* DFLAT
* FOCUS
* STUTTERED
* CBP

and these observation types are not enforced since it is possible to take them in modes where the headers are deliberately incorrect:

* BIAS
* DARK
* INDOME

Whether to fix a calibration header depends on the calibration team deciding that they need to use that observation.

## Timelines

For embargoed on-sky data it is preferable that a header fix is in place and merged before the data are unembargoed.
During unembargo the raw files are combined into Zips and the header information is also written to the zip.
Additionally the files start to propagate to remote sites and any headers that affect butler exposure metadata will be harder to correct since a targeted correction script will have to be run at each data facility.
Any header corrections that affect processing but do not affect Butler metadata have a more flexible timeline since we only have to have the correction in place prior to data processing beginning.

It is not possible to fix a header such that alert production can be corrected.

## References

```{bibliography}
```
