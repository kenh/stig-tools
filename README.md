# stig-tools

These are a collection of simple, command-line tools for manipulating
STIG checklist files.  I am making them public in the hopes that others
will find them useful.

I am aware of a few software packages that do STIG checklist management,
but the ones I have seen are all large applications with a web front-end
and at least one SQL database as a backend.  The thought of having to
fill out **MORE** STIG checklists to run STIG checklist management software
fills me with nausea and dread, so at our site we have gravitated to
a set of simple tools that remove as much of the manual drugery that
comes with dealing with STIG checklists.

## Overview

All of these tools operate on STIG checklist (`.ckl`) files, which
are created by tools like the DISA STIG Viewer from the XCCDF files
found in the actual STIGs.

While the XCCDF files have a public schema and seem to be reasonably
documented, the STIG checklist schema does not have any public
documentation I could find.  But in my experience everyone wants the
checklist files as part of the accreditation process, so our workflow
has generally been to use tools like the STIG Viewer to convert the
XCCDF files to checklist files, and then everyone does there work
directly on checklist files (I'm aware that the idea is the XCCDF files
are supposed to be used by SCAP validation tools, but in our experience
those tend to be hit and miss in terms of whether they work or not or
generate so many wrong answers that the results aren't so useful).

In general we find these tools to be useful after you have generated
the initial set of STIG checklists and need to upgrade to a newer
version of the checklist or make bulk edits based on feedback you
have received as part of the accreditation process.

These tools are all written in Perl and require the LibXML module;
`checktool` also requires the Perl YAML module.

### checktool

`checktool` is designed to take an existing checklist and migrate the
answers to a new version of the checklist; it can optionally take a
YAML file to fill in some of the answers to checklist items.

`checktool` has the following subcommands:

- `checktool check old-checklist new-checklist`

    This will give a report on the differences between `old-checklist`
    and `new-checklist` and inform you of the following differences:

    - Vulnerabilities (V-numbers) added to `new-checklist`
    - Vulnerabilities removed from `new-checklist`
    - Vulnerabilities that have changed between `old-checklist`
      and `new-checklist`.

      The concept of "changed" is relatively simplistic; `checktool`
      simply does a string comparison on each element in a vulnerability
      between the two checklists; if there are any differences the
      vulnerability is marked as changed (the element that has changed
      is reported).  As changes occur in the STIGs and the STIG
      tool, there are elements that are always different between two
      checklists and `checktool` has a list of elements to ignore
      for comparisons.  This list may need to be adjusted based on
      future revisions of the STIG tool; if you run into a situation
      where every vulnerability is being marked as changed, you
      may need to add to the list of elements in the `@ignoreattr` array.

- `checktool import old-checklist new-empty-checklist output-checklist [finding-details]`

    This will take an existing checklist (`old-checklist`) and a new
    checklist (`new-empty-checklist`) which should be empty and write a
    new checklist to `output-checklist`, which contains all of the data
    from `old-checklist`.  This includes all of the asset data (things
    like the hostname, MAC address, IP address, role, et cetera).  For
    vulnerabilities the data copied includes Finding Details, Comments,
    and Status.  If a vunerability has changed (see explanation under `check`)
    then the Finding Details and Comments will be copied over but the
    Status will be set to 'Open'.

    What happens internally is `new-empty-checklist` is used as the
    the basis for the output file, and all relevant data from `old-checklist`
    is inserted into the XML structure for `new-empty-checklist` and that
    is then written as `output-checklist`.  So if there is existing data
    in `new-empty-checklist` then it will either be overwritten by
    data from `old-checklist` (where data exists in `old-checklist`) or
    copied verbatim into `output-checklist`.

    `finding-details` is an optional argument where you can specify a
    YAML file that can specify vulnerabilities and Finding Details and
    Status fields; if `checktool` finds a vulnerability in the YAML
    file it will use that data in preference to the vulnerability data
    in `old-checklist`.  The idea behind this is if you have some Finding
    Details which are based on running commands on a host, you can use
    an external tool to generate the results of those commands in a YAML
    file and use that to populate the new checklist when you do an import.
    At this time we do not provide a tool to generate this YAML file.

    The YAML format is relatively straightforward; here's an example of
    what it would look like:

        ---
        V-123456:
          FINDING_DETAIL: |+
            Not a finding -- expected results  
            # grep -i Foo /etc/foo.config
            Foo: disabled  
          STATUS: NotAFinding
        V-456789:
          FINDING_DETAIL: |+
            Not a finding -- expected results  
            # grep -i Bar /etc/bar.config
            Bar: enabled  
          STATUS: Not_Reviewed

- `checktool legacyimport [import options]`  

    This will act the same way as `import`, but with one change.  The
    vulnerabilities will be matched in `new-empty-checklist` based on the
    legacy identifier for the vulnerabilities.  This is to be used when
    DISA renumbers the vulnerabilities between STIG releases (as of this
    writing, this is underway).

- `checktool dump checklist XPath-expression`

    This will dump all of the entries which match `XPath-Expression`
    in `checklist`.  This is only intended for developers.

### checkedit

This is a simple editor, a la `sed`, for checklists.  The idea behind
this is you can easily make simple changes to one or more checklists.
The usage statement built into `checkedit` has more details, but
the key options are:

- `--target ASSET-NAME VALUE`

    Set the target information `ASSET-NAME` to `VALUE`.  `ASSET-NAME`
    is normally something like `fqdn`, `ipaddr`, `mac`, et cetera.

- `--etarget ASSET-NAME EXPRESSION`

    Edit the value of ASSET-NAME using the Perl expression `EXPRESSION`.
    In the expression `$_` is set to the current value of `ASSET-NAME`;
    the expression should set `$_` to the desired value.

- `--ptarget ASSET-NAME`

    Print the current value of `ASSET-NAME`.

- `--vuln VULN-NUMBER VDATA VALUE`

    Change the value of `VDATA` in vulnerability `VULN-NUMBER` to the
    value `VALUE`.  `VDATA` could be things like `details`, `comments`,
    or `status`.

- `--evuln VULN-NUMBER VDATA EXPRESSION`

    Edit the value of `VDATA` for vulnerability `VDATA` using the
    Perl expression `EXPRESSION`. All of the rules from `--etarget`
    apply.

- `--pvuln VULN-NUMBER VDATA`

    Print the value of `VDATA` from vulnerability `VULN-NUMBER`.

Here are some sample things you could do with `checkedit`.

- It turns out different people used different format for MAC addresses, and
  your accreditors have requested you standardize on the `XX-XX-XX-XX-XX-XX`
  format.  To convert all checklists to that format, you could run:

        checkedit --etarget mac 'tr/a-f:/A-F-/' *.ckl

- You have received guidance that vulnerability V-987654 should be marked
  'Not Applicable' due to an approved waiver.  To change the all of the
  checklists you can run:

        checkedit --evuln V-987654 status na --vuln V-987654 details 'Not applicable due to authorized waiver; see reference XXXX' *.ckl

### copycomment

At one time we received guidance that all of the information in Finding
Details should also be in the Comments (that guidance was later recinded).
This was a simple program I wrote to perform this task.  I include it to
provide a simple example of how to use LibXML to modify information in
a checklist.

## Author

* **Ken Hornstein** - [kenh@cmf.nrl.navy.mil](mailto:kenh@cmf.nrl.navy.mil)
