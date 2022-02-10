# Patient-Facing EHI Export API

## Overview
The ONC Final Rule regulating the 21st Century Cures Act requires certified EHR systems to be able to provide computable copies of a patient's electronic protected health information (EHI) [^1]. Once the rule is fully in effect, this data set must encompass the entire clinical record (essentially: the "designated record set" that individuals are entitled to under HIPAA's right of access). Implementing a standards based API to access this information integrated with the existing regulatory mandated patient-facing FHIR APIs that encompass the USCDI data set would empower patients to leverage apps across all of the data in their medical record.

## Use Cases
There are many potential use cases, including support for apps that enable patients to share complete records with a provider for a virtual second opinion or to transition care, apps that share a subset of a patient's data elements with a research study (today, this often needs to be done manually by investigators through broad records requests and faxed data), and apps that enable patients to review their own health record data to identify errors and explore trends.

For example, a recently developed shared decision making app around lung cancer screening requires a handful of data elements not currently available in the USCDI data set. If an EHI API was broadly available, the app could leverage a single patient-facing SMART on FHIR flow to enable users to choose their healthcare provider and authenticate through the provider's patient portal to authorize access, and then populate the risk assessment calculation with both standard data elements available through USCDI, and non-standardized data elements drawn from the EHI data set. 

## Benefits 
Although the data included in USCDI will expand over time, it seems likely that the growing scope of health-related data (e.g., genomic data, microbiome data) will continue to outpace the development of detailed standards. Furthermore, variation in clinical practice will continue to drive a "long tail" of unstructured data. While working with vendor specific EHI data sets will require more effort than working with well structured USCDI data, vendor documentation requirements in the ONC rule should make it possible for the community to collaborate on open source libraries, "a small price to pay for gaining access to a much larger set of patient information many years earlier than would otherwise be possible"[^2]. 

An end-to-end API would offer a modern and convenient extension to an EHI implementation where, for example, a patient may otherwise need to obtain their data by by phoning the medical records department and being mailed physical media. Even a download button in the patient portal presents user experience challenges since portals experiences vary, making features difficult to find and correctly describe (e.g., if a third party is trying to guide patients toward the export functionality in a variety of portals). This was a clear challenge for anyone trying to identify the "Transmit to a third party" features of a patient portal in the MU2 time frame. Further, managing files may be challenging on many patient devices like mobile phones and tablets, and some files may be best suited for off-device cloud storage [^3]. 

Enabling people to move and share their data without requiring them to take on cumbersome and complex file management tasks has long been a goal among technology companies and is the impetus behind the joint Apple, Facebook, Google, Microsoft, and Twitter data transfer project [^4].

## Authentication and Authorization
An EHI API could leverage the SMART App Launch Authentication and Authorization flow necessary for EHR certification under the 21st Century Cures Act ONC regulation [^5]. An app wishing to obtain a copy of a patient's EHI would request a relevant scope (perhaps something like `patient/ehi-export`). In the authorization screen presented to a patient as part of the SMART flow, the patient could choose to approve or deny the sharing of their EHI with the app. As with other SMART App Launch authorizations, if the authorization is granted to provide ongoing access, the patient can subsequently revoke this authorization through the patient portal. 

A more granular scope scheme could also be developed and may prove advantageous. This would enable apps to request access to a subset of the available EHI, for example, demographic data and medical images, but not other clinical data. Such an approach allow patients greater control over the data they're providing to an app, but also increases implementation complexity and would differ from manual EHI requirement implementations making it inadvisable in the initial version of an API.

## Asynchronous FHIR Operation
The EHI request API could be implemented as a new FHIR operation that builds on the FHIR Bulk Data Access IG ("Bulk FHIR") [^6] with a request endpoint like `/Patient/:id/$ehi-export`. As with other Bulk FHIR requests, processing would be asynchronous. The request would return a job identifier rather than data, and it would be the responsibility of the app accessing the API to check for completion (though the API  provides a mechanism for FHIR servers to provide hints to the app on expected completion times). 

API parameters that limit the data being returned could also be incorporated into the operation's design as optional parameters. While this may be beneficial to both client and server applications by reducing the size of the data set being transmitted and parsed, it would also require community consensus and increase implementation complexity.

On the backend, healthcare institutions could use automated systems to prepare the EHI data set either as component of their EHR, or as an aggregator service that extracts data from multiple systems and consolidates the results. These systems could even integrate with a workflow platform to incorporates manual steps, such as HIM department review to verify the validity of the request, or enabling a user at the health system to attach files to the response from systems that do not currently offer an API. 

The Bulk FHIR manifest returned by the completed job would, in addition to including FHIR resources with the patient's data in the structured FHIR format where available, also include FHIR `DocumentReference` resources that embed or provide urls pointing to files in vendor-defined formats (e.g., vendor-specific JSON formats or CSV database table snapshots). An informal survey of EHR vendors by the Argonaut FHIR accelerator in 2021 indicated vendor plans to provide EHI data in SQL, CSV, and JSON formats and the flexibility provided by this approach should address the needs of real world implementations.

The `DocumentReference` resources can also incorporate metadata about each file, such as a link to the corresponding data dictionary and documentation in the description element, a vendor defined identifier for the data being provided in the type element, and the file's data format in the attachment element's `contentType` element.

The completed export result might look like:
```json
{
  "transactionTime": "[instant]",
  "requiresAccessToken" : true,
  "output" : [{
    "type" : "Patient",
    "url" : "http://serverpath2/patient.ndjson"
  },{
    "type" : "Observation",
    "url" : "http://serverpath2/observations.ndjson"
  },{
    "type" : "DocumentReference",
    "url" : "http://serverpath2/csv_export.ndjson"
  }],
  "error" : [{
    "type" : "OperationOutcome",
    "url" : "http://serverpath2/err_file_1.ndjson"
  }]
}
```

... where the document_references.ndjson file would contain items like:

```json
{
  "meta": {
    "tag": [{ 
        "code": "ehi-export",
        "display": "generated as part of an ehi-export request"
    }]
  },
  "resourceType": "DocumentReference",
  "description": "Demographic information not included in Patient resource, described at http://vendor.com/docs/cures-ehi-demographics.html",
  "category": [{
    "coding": [{"system": "http://ehr.example.org/docs/ehi/v2.0.1", "code": "demo-table", "display": "Demographics table export"}]
  }],
  "content": [{
    "attachment": {"url": "http://server.example.org/patient_file_1.csv", "contentType": "text/csv"}
  }],
  "status": "current"
}
```
[^1]: https://www.healthit.gov/test-method/electronic-health-information-export
[^2]: William J Gordon, Daniel Gottlieb, David Kreda, Joshua C Mandel, Kenneth D Mandl, Isaac S Kohane, Patient-led data sharing for clinical bioinformatics research: USCDI and beyond, Journal of the American Medical Informatics Association, Volume 28, Issue 10, October 2021, Pages 2298–2300, https://doi.org/10.1093/jamia/ocab133
[^3]: https://github.com/jmandel/interop-2019-nprms/blob/master/ehi-export.md 
[^4]: https://engineering.fb.com/2019/12/02/security/data-transfer-project/
[^5]: http://hl7.org/fhir/smart-app-launch/app-launch.html
[^6]: https://hl7.org/fhir/uv/bulkdata/export.html
