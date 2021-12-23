# Epic Application Developer Survey Collateral

In my role as Director of Pathology Informatics I have unrestricted SQL access to a number of EHC’s databases including:

[a] our Operational Data Store – a replicated Cerner Millennium schema that includes our clinical pathology LIS and blood bank information system that is analogous to Epic Clarity,

[b] our Clinical Data Warehouse – a dimensional schema backing our Microstrategy BI tool that is analogous to Epic Caboodle, and

[c] Cerner CoPathPlus – our anatomic pathology LIS which will be replaced with Epic Beaker AP.

I rely on this SQL access daily to do lots of things. I am generally operating from Java or Python platforms, although sometimes I'm just using an ad-hoc query tool (eg, SQL*Developer). In the interest of brevity, I include some examples from public GitHub reposistories that are easy to link to:

[a] I inspect laboratory test build – the connect-by query at the end of https://github.com/ghsmith/eshViewer/blob/master/src/main/java/eshviewer/NormalizedHierarchyNodeResource.java runs in our ODS and normalizes our hierarchical result-to-order namespace (event_set - event_code - discrete_task_assay - primary_mnemonic - synonym) so that we can efficiently ask complicated questions in this domain.

[b] I query information into XML schemas to support various laboratory quality initiatives – the XML schema at https://github.com/ghsmith/CoPathDump/blob/master/src/main/resources/CoPathDump.xsd is populated from our CoPath database and used as part of a radiology-pathology correlation exercise),

[c] I routinely make use of correlated and non-correlated inline SQL, SQL analytical functions (eg, dense_rank keep first), SQL aggregation functions (eg, listagg) – the query in https://github.com/ghsmith/export4dj/blob/master/src/main/java/edu/emory/pathology/export4dj/finder/PathNetResultFinder.java runs in our CDW and demonstrates some of these features

[d] I support batch-integrations (eg, flat file) with some of our specialized laboratory information systems – the class at https://github.com/ghsmith/CoPath2Go/blob/master/src/main/java/edu/emory/c2g/CreateTSO500GoManifest.java accesses the CoPath database to create a flat-file case manifest for our GenomOncology next-generation sequencing annotation system.

[e] I also routinely need to query laboratory data with other clinical data (eg, diagnosis codes, procedure codes, demographics) in creative ways to support clinical laboratory operations – I don’t have any of these in public git repositories so I've added one to this "stuff" GitHub repo and linked to it here: https://github.com/ghsmith/stuff/blob/main/example01.md . This returns result encounter and, if applicable, any active inpatient encounter for SARS-CoV-2 positive patients that have qualifying B-cell malignancy diagnosis codes.
