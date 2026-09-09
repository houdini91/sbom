.. SPDX-License-Identifier: CC-BY-4.0

.. _chapter-appendix:

Appendix
========

External SBOM Metadata
----------------------

This document strongly encourages vendors to embed the SBOM metadata into the respective binaries, but there are two situations where externally referenced SBOM metadata would be allowed:

- Where the binary is loaded onto critically space-constrained devices, for example microcode that is loaded into the processor itself.
- Where only later newer versions of the component have embedded SBOM metadata, and backwards compatibility is required with older revisions.

In these cases, the *component vendor* **MUST** provide “detached metadata” from the same source (or in the same archive file) as is used to distribute the immutable blob.

As the SBOM metadata is detached, vendors **MUST** ensure that the files do not get “out of sync” and are updated at the same time in the firmware source tree.
Detached metadata **MUST** `always contain the SHA256 hash value of the binary <https://github.com/hughsie/python-uswid/#use-cases>`_ as evidence to allow validation and **MAY** be signed using a detached signature if the archive is not already signed.
The public key **SHOULD** be distributed on a keyserver or company website for verification.


What a Hash Covers
------------------

A hash says nothing about what was done to the bytes before it was taken.
Usually nothing is, but some *components* are transformed between being built and being shipped: a PE binary placed into an EFI file volume has base relocations applied throughout the image, and one converted to the smaller Terse Executable (TE) format has its PE/COFF headers replaced by a TE header.
Neither changes what the *component* does, but both change its bytes, so a hash taken at build time and one re-derived from a shipped image can differ when nothing is wrong.

This section concerns hashes of a binary.
The *source code* file hash and tree hash required in :ref:`chapter-metadata`, and the checksum of a generated SBOM used as a collection ID, are unaffected.

Where an SBOM records a hash of a binary and does not say what was hashed, that hash **MUST** be of the binary as distributed, with no transformation applied.

A *component vendor* or *firmware vendor* **MAY** also publish a hash taken over a transformed form of the same binary, so that it can be compared against a build-time value.
Such a hash **MUST** accompany the untransformed hash rather than replace it, and **MUST NOT** be recorded in a coSWID ``hash-entry``, a CycloneDX ``hashes`` entry or an SPDX ``checksums`` entry.
Those fields mean the hash of the file, so a tool that does not implement the transformation would read one from them and report a mismatch for a binary that is not modified.
The value **MUST** therefore identify the transformation itself, and **SHOULD** be a URI, following the pattern already set by ``gitoid:blob:sha256:…`` and ``swh:1:cnt:…``.
In CycloneDX it is carried as a *component* property.
SPDX has no field defined for it: ``ContentIdentifier`` is the closest, but its type vocabulary is closed to ``gitoid`` and ``swhid``, so a third value there does not validate.
A value that identifies itself survives that gap, which is why the label belongs in the value rather than in a field:

::

  {
    "name": "osf:normalizedHash",
    "value": "uefi-pe-rebase0.v1:sha256:1348ff9c695f80b3..."
  }

This document defines no such transformations.
The property name above is illustrative, and naming follows the `CycloneDX property taxonomy <https://github.com/CycloneDX/cyclonedx-property-taxonomy>`_; an ``osf`` namespace would first need registering there.

A tool **MUST NOT** compare hashes produced by different transformations, an absent label meaning none was applied.
Such hashes are not comparable, which is neither a match nor a mismatch, and a tool **MUST NOT** report a binary as verified on that basis; an unlabeled hash that does not match is a mismatch.


Wasted Space Concerns
---------------------

Some vendors have expressed concerns about “wasted” space from including the SBOM data in the binary image.
For source components such as CPU microcode, a single *component* and vendor *entity* would use an additional ~350 bytes (zlib compressed coSWID), compared to 48kB for the average EFI binary and 25kb for a typical vendor BGRT “splash” logo.

The ``uswid`` command can automatically `generate <https://github.com/hughsie/python-uswid#generating-test-data>`_ a complete “worst case” platform SBOM with 1,000 plausible components.
This SBOM requires an additional 140kB of SPI flash space (uncompressed coSWID), or 60kB when compressed with LZMA.
For reference, the average free space in an Intel Flash ROM BIOS partition is 5.26Mb, where “free space” is defined as a greater than 100KiB stream of consecutive 0xFF’s after the first detected EFI file volume.
Adding the SBOM as embedded metadata would use 1.1% of the available free space.
Other firmware ecosystems such as Coreboot also `now include SBOM generation <https://doc.coreboot.org/sbom/sbom.html>`_ as part of the monolithic image.


Getting the Runtime SBOM
------------------------

The ACPI ``SBOM`` ACPI table may be used in the future to return the coSWID formatted binary SBOM data from any device exporting an ACPI callable interface.
Further details will be provided when the SBOM table has been implemented.

If the platform allows direct access to the system SPI device, then the entire firmware image can be dumped to a local file and analyzed by tools such as ``uswid``.

Converting the SBOM
-------------------

The embedded SBOM **SHOULD** be converted it into one or more SBOM export formats before publication.

This can be achieved easily using tools such as ``uswid``.
For example, this can be used to produce two JSON files in CycloneDX and SPDX formats from the platform image:

::

  $ uswid --load rom.bin --save cyclonedx-bom.json
  $ uswid --load rom.bin --save spdx.json

Signing the SBOM
----------------

The embedded SBOM **MAY** be signed, and **MAY** also be included in the firmware checksum.
If the firmware component is signed then the SBOM **SHOULD** be included in to the signature.
The signing step is optional because a malicious silicon provider can typically do much worse things (e.g. adding or replacing a DXE binary) than modify the SBOM metadata.

Using the LVFS
--------------

When firmware is uploaded to the LVFS it automatically extracts all available SBOM metadata and generates `a HTML page <https://fwupd.org/lvfs/devices/component/64327/swid>`_ with SPDX, SWID and CycloneDX download links that can be used for compliance purposes.
The LVFS **MAY** allow vendors to upload firmware or platform SBOMs without uploading the firmware binary.
Other services like Windows Update may offer this service in the future.

The VEX "trusted neutral entity" **MAY** also be the LVFS, even for firmware updates not distributed by the LVFS.
Uploading VEX data requires vendors to register `for a LVFS vendor account <https://lvfs.readthedocs.io/en/latest/apply.html>`_ which is available at no cost.
