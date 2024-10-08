# templayManifest-common
This repository describes the JSON format and rationale. For implementations, see [teamplayManifestR](https://github.com/dnebdal/teamplayManifestR) and [teamplayManifest-py](https://github.com/dnebdal/teamplayManifest-py).

The [teamplay digital health platform](https://www.siemens-healthineers.com/no/digital-health-solutions/teamplay-digital-health-platform) from Siemens Healthineers can be used to run containers written elsewhere for the purpose of running some sort of analysis on your own data. As designed, it expects images following the DICOM standard, containing all required metadata embedded in the image. This does not work when extending the platform to work with other data formats, like the text files typical of other -omics. It also seemed reasonable to use the same format to describe the files created by the analysis and returned to the user.

This repository documents a suggested format for manifest files that can accompany those files, containing the neccessary metadata.

# Workflow
The intended flow is something like this:

- End user: Create a manifest to describe their files
- End user: Package the input files and manifest as a zip file
- Teamplay middleware: Parse the manifest to route it to the right analysis container
- Analysis code: Parse the manifest to get a list of input files and their descriptions
- Analysis code: Update the manifest to mark it as done and describe the files produced
- Analysis code: Package the output files and manifest as a zip file

# HL7 FHIR
For interoperability reasons, the manifest format conforms to [HL7](https://en.wikipedia.org/wiki/Health_Level_7) [FHIR](https://en.wikipedia.org/wiki/Fast_Healthcare_Interoperability_Resources), version 5.0.0. This format is thoroughly documented over on [their website](http://hl7.org/fhir/). Specifically, a manifest is a [Task](https://www.hl7.org/fhir/task.html), and the files are described as [Attachments](https://www.hl7.org/fhir/datatypes-definitions.html#Attachment).

The specific details of how to translate their format descriptions into JSON are not completely obvious - I have spend a lot of time guessing and running my JSON through the [FHIR Validator](https://validator.fhir.org/). Remember to set the version to 5.0.0 in the Options tab.

# Format examples
The manifest will be read at a few different stages:
- By Teamplay, to route the data to the right analysis container
- By the analysis container to get a description of the input files
- By who or whatever consumes the output package from the analysis container

In the format descriptions, I use dots for nested fields: 
Given `"id": {"reference":"value"}` , `id.reference` is set to "value". 

## Common header
Some fields must always be included:

- `resourceType` is always `Task`
- `text` is a human-readable description of the task
    - `text.status` doesn't matter bust must be one of the valid values; I suggest `generated`
    - `text.div` is a HTML div element containing the description
- `status` is either `requested` or `completed`, depending on the stage
- `intent` doesn't matter but must be one of the valid values, I suggest `order`
- `authoredOn` is when the manifest was first generated. Must be a FHIR [dateTime](https://www.hl7.org/fhir/datatypes.html#dateTime).
- `focus.reference` is the patient/sample id
- `for.reference` is the name of the zip file
- `encounter.reference` is the time point the sample was collected at, as free text
- `requestedPerformer` must be an array, but we only use the first element
    - `requestedPerformer[0].reference.reference` is the identifier for the container/analysis to run

This example is a manifest for a sample called "OUS_Patient1", gathered at "Start of Treatment", to be analysed on "OUS-OMICS-PANC",
and packed into zip file `OUS_Patient1.Start_of_treatment.OUS-OMICS-PANC.1710166715.zip`.

```
{
  "resourceType": "Task",
  "text": {
      "status": "generated",
      "div": "<div xmlns='http://www.w3.org/1999/xhtml'>Input task for OUS-0001, created 2024-02-28T13:45:37+01:00<\/div>"
    },
  "status": "requested",
  "intent": "order",
  "authoredOn": "2024-02-28T13:45:37+01:00",
  "focus":{"reference":"OUS_Patient1"},
  "for":{"reference":"OUS_Patient1.Start_of_treatment.OUS-OMICS-PANC.1710166715.zip"},
  "encounter":{"reference":"Start of treatment"},
  "requestedPerformer" : [{
      "reference":{"reference":"OUS-OMICS-PANC"}
    }]
}
```

## File descriptions

To provide information about the files included, add an `input` or `output` block to the
manifest, depending on if you are a user sending data into teamPlay (`input`), or 
a container inside teamPlay packaging your results (`output`). I suggest leaving the input block
in the manifest, as a record of which files were used.

Every single field of [Attachment](https://www.hl7.org/fhir/datatypes.html#Attachment) is optional. We use three:
- `type.text` describes what kind of data is in the file - which omic, in effect.
- `valueAttachment.contentType` is the [MIME type](https://en.wikipedia.org/wiki/Media_type), it describes the *format*. 
- `valueAttachment.url` is the file name. Since this is an URL, it needs a protocol; use "`file://`".

It's obviously important which values are allowed in these fields. In practice this is up to whoever wrote the container 
that will receive the manifest - it would be good to agree on a standardised list.

For the MIME types, try to stick with something from the official [registry](https://www.iana.org/assignments/media-types/media-types.xhtml).
At a guess, these ones will cover most uses:
```
application/dicom           DICOM dcm file
application/pdf             PDF
application/octet-stream    Any binary format without a more specific type, e.g. BAM, RDS
image/jpeg                  JPG/JPEG
image/png                   PNG
image/tiff                  TIFF
text/csv                    CSV (comma-separated; do not use semicolon-separated)
text/plain                  Any unstructured text, like FASTQ
text/tab-separated-values   Any tab-separated text file, including VCF
```

The `input` and `output` blocks are arrays of these descriptors - if you only have one file, it still needs to be an array of length one.

This example input block describes two files: A VCF file of mutations, and a CSV file of methylation values.
```
  "input": [
    {
      "type": {
          "text": "Mutation"
        },
      "valueAttachment": {
          "contentType": "text/tab-separated-values",
          "url": "file://mutations.vcf"
        }
    },
    {
      "type": {
          "text": "Methylation"
        },
      "valueAttachment": {
          "contentType": "text/csv",
          "url": "file://methylation.csv"
        }
    }
  ]
```
## DICOM reference
Instead of an input file, the input can come from a DICOM node. To reference this, store a text ID in a valueReference, with the exact type text from the example 
(it's a reference to the [SNOMED](https://browser.ihtsdotools.org/?perspective=full&conceptId1=1119411007&edition=MAIN/2024-08-01&release=&languages=en) definition).
```
"input": [
    {
      "type": {
        "text": "Digital Imaging and Communications in Medicine module (core metadata concept)"
      },
      "valueReference": {
        "identifier": {
          "value": "imaging_study_uid_1"
        }
      }
    }
  ]

```


## Output-specific fields

To convert a manifest from "a task to be run" to "a description of results" only takes a few changes.

- Set `status` to `completed`
- Optionally change the `text.div`
- Add a `lastModified` datetime
- Add an `output` block describing the files created

Here is a complete output manifest, matching the input manifest above:

```
{
 "resourceType": "Task",
  "text": {
      "status": "generated",
      "div": "<div xmlns='http://www.w3.org/1999/xhtml'>Results for task for OUS-0001, created 2024-02-28T14:00:00+01:00 <\/div>"
    },
  "status": "completed",
  "intent": "order",
  "authoredOn": "2024-02-28T13:45:37+01:00",
  "lastModified": "2024-02-28T14:00:10+01:00",
  "focus":{"reference":"OUS_Patient1"},
  "for":{"reference":"OUTPUT.OUS_Patient1.Start_of_treatment.OUS-OMICS-PANC.1710167000.zip"},
  "encounter":{"reference":"Start of treatment"},
  "requestedPerformer" : [{
      "reference":{"reference":"OUS-OMICS-PANC"}
    }],
     "input": [
    {
      "type": {
          "text": "VCF"
        },
      "valueAttachment": {
          "contentType": "text/tab-separated-values",
          "url": "file://mutations.vcf"
        }
    },
    {
      "type": {
          "text": "Methylation"
        },
      "valueAttachment": {
          "contentType": "text/csv",
          "url": "file://methylation.csv"
        }
    }
  ],
  "output": [
    {
      "type": {
          "text": "Survival report"
        },
      "valueAttachment": {
          "contentType": "application/pdf",
          "url": "file://survival_report.pdf"
        }
    },
    {
      "type": {
          "text": "Survival table"
        },
      "valueAttachment": {
          "contentType": "text/csv",
          "url": "file://survival_table.csv"
        }
    }
  ]
}
```

# Zip file
The names of the zip files created by the Python and R tools have a specific layout:

`[NEW|RES].sampleID.encounter.containerID.timestamp.zip`

Thus, `NEW.OUS-Patient-0001.EOT.OUS-OMICS-PANC.1712753272.zip` is a file 
- Uploaded to teamplay: `NEW`
- For sample `OUS-Patient-0001`
- Taken at timepoint `EOT`
- To be analyzed in container `OUS-OMICS-PANC`
- Created at UNIX timestamp `1712753272` (2024-04-10 12:47:52 GMT)

If everything goes well, the results should appear in a file called something like `RES.OUS-Patient-0001.EOT.OUS0001.1712753400.zip` some time later.

# Other tools
## jq
While the R and Python packages should cover most uses, it is also possible to work with the manifest files with any JSON aware tool.
For instance, [jq](https://jqlang.github.io/jq/) makes it reasonably easy to query JSON files from the command line. To extract the requestedPerformer from a manifest, this works fine:

`jq '.requestedPerformer[0].reference.reference' MANIFEST.json`

# HL7 validator
The HL7 organisation maintains a [command line validator](https://github.com/hapifhir/org.hl7.fhir.core), in addition to the web version. It can be downloaded directly from [here](https://github.com/hapifhir/org.hl7.fhir.core/releases/latest/download/validator_cli.jar) (Java JAR file), and run with `java -jar validator_cli.jar MANIFEST.json`.
