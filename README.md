Retention business requirements There are two types of retention: Internal Retention and Compliance Retention

Internal Retention policy (NO WORM flag set) will allow free modifications of document properties and binary files as long as binary file or document itself is not removed (BLOB can even be replaced). Users can add annotations and edit meta attributes. Only Delete operation is locked. Everything else is allowed. Version management of document is enabled and works as usual, no additional customizations are needed comparing to non-retained document.

Compliance Retention is a state of document where any modifications to the main content blob, metadata and visual annotations made on top of it are forbidden. There are some exceptions related to ECM version upgrade in regards of metadata modifications that are documented below. Versioning of the retained document is disabled in this mode.

Examples:

Internal Retention: Document could be viewed/edited/annotated for blob and metadata. Blob can't be replaced. Only remove blob and remove document operations are locked. Document can still be versioned. Modification of document should trigger a document version change.

Compliance Retention: Document could be viewed but not edited or annotated, No changes in metadata properties are allowed except related to ECM upgrade. Removing document, blob are also not allowed. No versioning is supported any longer as document state is expected to be unmodifiable.

In internal retention mode, all metadata properties of the retained document could be modified.

In compliance mode, metadata properties of document are a subject of a full retention the same as main blob and follow the same read only guidelines. However they could be modified on the certain scenarios (see bellow). Reason for this is that metadata properties are dependent on the version of ECM application and might be changed with it. For example Nuxeo is not the only ECM solution, that Fidelity might use in the coming years and migration of document metadata properties to another version/ECM platform is theoretically possible and even likely (e.g HXP). Also future versions of Nuxeo ECM platform might require changes in document metadata done for compatibility reasons. And what definitely will happen is that fidelity project team extends or alters the document metadata based on specific requirements provided in the future. Some or all of these events might happen during the period of retention. So it is not realistically possible to lock metadata properties from being altered, removed or added to the document under retention and it should not be expected. However even it is not possible to fully ensure unmodifiable state of metadata it should be treated as such for all users including admins and REST services. All modifications to metadata of retained documents (WORM storage) must be only done for sake of upgrading version of ECM software or extending its functionality by deploying new fidelity custom packages. All modification done for any other purpose should be restricted.

Government requirements demand that content of the main blob when under compliance retention must be searchable and downloadable as well as its annotation. In order to facilitate this process Nuxeo should make every effort possible to provide indexing of content of main blob and when main blob has no indexable content (unsupported format) a custom logic should be provided to index the content. If this is not possible due to the nature of the document, then metadata of the document that is locked and stored in a read only mode will be used in order to ensure document is still searchable by metadata, however changes to document metadata done due to the version upgrades and other product customization should be allowed when necessary to keep document storage consistent (see above).

Documents under retention could be treated differently if additional flags are attached to the retention policy that matches to the document retention label. If a total number of flags used by fidelity is currently unknown and can be changed at any time, the most important requirement at this moment is that documents that have retention rule with WORM storage flag to be stored in a separate WORM (write once read many) S3 bucket or an alternative solution provided by another cloud vendor. If a document retention rule does not have WORM flag set then document if under retention is stored in the same S3 bucket where it would normally be stored and its non-removable state should be assured by application. Rules with WORM and w/o WORM flag could coexist with each other - some documents under retention should go to WORM storage and some don't. Current product version does not support this logic and based on the server configuration all documents can either go to a WORM bucket if under retention or none of them will irrespectively of the retention rule design flag. It has to be corrected.

Possible that documents in WORM storage have to be deduplicated the same as on a regular storage. BLOB has to be stored there as long as there is at least one document that referring to it. Right now, no deduplication of documents in WORM storage is supported by product and it has to be addressed as client indicated it as a problem of increased storage cost (more data stored than necessary).

Fidelity team will internally discuss if deduplication is required for WORM storage.

Example:

Document A and Document B use the same file ShareHolderDoc1.pdf. Document A goes on retention on 02/15/2022 , Document B goes on retention on 02/15/2023. Both documents have 2 years retention policy. With deduplication: Only one copy of document ShareHolderDoc1.pdf will exist from 02/15/2022 till 02/15/2025. With no deduplication: One copy exists from 02/15/2022 till 02/15/2023 (Document A is under retention), Two copies exist from 02/15/2023 till 02/15/2024 (Doc A and B both on retention One copy exist from 02/15/2023 till 02/15/2025 (only Doc B on retention). Documents that stored in WORM storage should have the main blob and annotations made to it be stored side by side. So if before going to retention a document had a visual annotation made in an electronic tool (e.g Nuxeo Enhanced Viewer) then these annotations should be stored in WORM storage and be unmodifiable. If WORM flag is not set on the retention rule level then annotations are a subject of modification the same as metadata properties. This feature is not supported by product and should be implemented. Right now product is unable to store annotations in WORM storage at all.

As an additional feature, it is recommended to make "no versioning/read only" policy configurable even for documents under internal retention. So even a document retention policy is set to not store documents in a WORM storage it is desired to have an option to be able to ensure documents "read only/no-versions status" using nuxeo system features. By default it should be disabled and only delete operation should be disabled for documents under internal retention when versioning should be turned on. But it is desired to be possible to set "No Version/Read Only" flag on Retention Rule level, that will make internally retained documents be read only but they will not be stored in WORM storage. This is how other ECM platforms work based on Fidelity feedback.

Custom operations for post-retention actions are also desired. There should be an extension point where a custom operation can be introduced that can be selected instead of Trash/Delete in a drop down for the post-retention action.

Fidelity thinks that it might be helpful to have a report tool that show documents that have their retention status change in the coming weeks or months. It probably can be done by professional services team.

Implementation flow Nuxeo product allows to create retention rules that define the number of years that the document must be on retention and a post retention action. There are some additional attributes that help to determine the type of retention rule. See below. (done)

Nuxeo allows to define a list of retention categories to match fidelity's current design. All retention categories will have a retention label property assigned and get a link to a product specific retention rule object (see above) that has a number of retention years and other attributes. Retention category object also would have a list of additional flags that fidelity finds useful (WORM storage or not)(done). There is an overlap in properties defined in retention category and retention rule objects but it is not a problem now. We should review it and eliminate duplicates (not done)

Each fidelity document should have a property of document category (e.g SP2007-ECOM, IX, CA etc) set or otherwise it would not be eligible for retention. This property will match some vocabulary value that fidelity has to provide. (not done)

Schema property fid:document_category (string/single value will be available to all documents.

A simple (not hierarchical) vocabulary mapping document category to a retention label VOC_DocCategories

An event handler that sets a retention label automatically when document is updated or created and document category value is different or newly set. Setting is done by a lookup into VOC_DocCategories. Folder documents excluded.

It is not allowed to change document category/retention label if document is under retention (event handler should throw an error if attempt is made and UI should have values shown as read only)

List of possible document categories is not limited and is updated all the time so potentially it is possible to have a new document category assigned to a document that does not have a mapping yet to a retention label defined in into VOC_DocCategories. In this case a warning should be shown on a document page. It is also possible to change mapping from document category to retention label by changing value in VOC_DocCategories and it should affect all the documents that are not currently under retention (see below)

A scheduled process searches for all non-folderish documents not under retention that have a retention label that does not match document category because mapping was changed (see above).

Example 1: There is a mapping of document category BX-8 to retention label 1870 Document A was created with document category BX-8 Label 1870 is assigned to document A Admin changes mapping of document category BX-8 to retention label 1871 The scheduled process changes value of retention label to 1871 to document A

Example 2: There is no mapping of document category BX-9 Document B was created with document category BX-9 No label is assigned to document B Admin changes mapping of document category BX-9 to retention label 1872 The scheduled process changes value of retention label to 1872 to document B

Example 3: There is a mapping of document category BX-8 to retention label 1870 Document C was created with document category BX-8 Label 1870 is assigned to document C Document C goes on retention Admin changes mapping of document category BX-8 to retention label 1871 Nothing happens to document C , it still has retention label 1870 Optional (A separate ticket down in the backlog – not urgent but good to have) Create a page provider that lets search for documents that have no retention label a) because a document category was not set b) because the selected document category has no mapping to a retention label.

Add filters by document type and date modified.

Add a reference to this page provider to admin/retention categories page where admin can easily locate such documents. On top of it UI should show a list of all the document categories that ARE USED but do not have a retention label assigned. This way admin can easily finds which values of document categories are needed to be mapped to retention labels inside VOC_DocCategories.

Each fidelity document should have a property retention label (fid:rcat) that defines a retention rule to be used for this document. Until this property is assigned to a document, it is ineligible for retention. (done)

There should be a process that updates the retention label of the document when a document category is changed or assigned. This process might use additional metadata properties to change/assign the retention label. This logic has to be specified by fidelity (not done). The document category value may be set directly from the UI using vocabulary values provided by the fidelity team (TODO/not done) or it could be set by a bulk import/update process. (not done)

There are two types of retention policies supported - event based and time based. (done)

Time based policy starts when a document has a retention label that points to a time based retention rule and attribute fid:retentionStartDate value is a valid date in the past. Retention end date is determined as retention start date plus number of years defined in retention rule(done)

Example 2:

Create a retention rule with label 1680 that is time based with 7 years of retention starting from the retention start date
Create a blank document without a retention label
Document is not under retention
Set retention label to 1680
Document is not under retention
Update fid:retentionStartDate to be tomorrow
Document is not under retention
Wait for tomorrow
Document is now under retention and its retention end date is set to current date plus 7 years
Event based retention starts immediately after an event based retention rule is assigned to a document or a document is created with retention label linked to an event based retention rule. However, retention end date stays undermined until value of fid:retentionStartEvent is populated by a value configured in retention category (for example TOA). When value of fid:retentionStartEvent is set, then the retention end date is calculated as fid:retentionStartDate (default to current date) plus number of years specified in retention policy. (partially done) Example 1:

Create a retention policy with label 22 that demands TOA event and 7 years of retention thereafter
Create a blank document with retention label 22
Document is now under retention and its retention end date is undetermined (not yet done)
Update fid:retentionStartEvent property of the document to value TOA 
Document is now under retention and its retention end date is set to current date plus 7 years (not fully done)
Example 2:

Create a retention policy with label 22 that demands TOA event and 7 years of retention thereafter
Create a blank document without a retention label
Document is not under retention
Set retention label to 22
Document is now under retention and its retention end date is undetermined (not done)
Update fid:retentionStartEvent property to value TOA and fid:retentionStartDate to yesterday 
Document is now under retention and its retention end date is set to yesterday plus 7 years(not fully done)
Retention specific operations done in manual mode must be only allowed to administrators. The list of operations should be very minimal and include:

setting retention start date property setting retention start event property extending retention date into the future Right now administrators are allowed to set retention labels for documents manually; this should not be allowed going forward. Administrators are also currently allowed to attach retention rules to the documents manually which should not be allowed. Retention rule assignment must be done by a scheduled task and scheduled task only. There will be no manual overwrite going forward.

As stated above documents should go on retention based on a value of retention label property that has to be set ONLY by a scheduled process using document metadata (e.g document category value). (not done). Until it is implemented we can use a manual process of assigning the retention label.
