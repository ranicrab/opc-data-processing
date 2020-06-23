# Common Processing Workflows

## Table of Contents  

* [Set up environment](#set-up-environment)    
* [Upload a new dataset with custom identifiers](#upload-a-new-dataset-with-custom-identifiers)   
* [Update a dataset](#update-a-dataset)  
* [Manually obsolete datasets](#manually-obsolete-datasets)  
* [Archive objects](#archive-objects)  
* [Setting access policy on individual objects](#setting-access-policy-on-individual-objects)  
* [Copy a package as a template](#copy-a-package-as-a-template)  
* [Search datasets with Solr](#search-datasets-with-solr)  

## Set up environment

This loads the 3 libraries used commonly, and sets up end points for the
coordinating node, member node, and a combination of the two (the D1Client class).
```
library(dataone)
library(datapack)
library(EML)

cn <- CNode("PROD")
mn <- MNode("https://opc.dataone.org/metacat/d1/mn/v2")
mn@env <- "PROD"
d1c <- D1Client(cn, mn)
```

## Upload a new dataset with custom identifiers

If it weren't for the custom identifiers, I would strongly recommend using the
web interface for this. Here is how to do it in R though. More thorough examples
can be found [here](http://training.arcticdata.io/materials/arctic-data-center-training/programming-metadata-and-data-publishing.html)


#### Create a valid EML document

See the R EML package [webpage](https://docs.ropensci.org/EML/) for more information.

```
me <- list(individualName = list(givenName = "Jeanette", surName = "Clark"))
my_eml <- list(system = "https://opc.dataone.org", packageId = "custom identifier" 
              dataset = list(
              title = "A Minimal Valid EML Dataset",
              creator = me,
              contact = me)
            )


write_eml(my_eml, "ex.xml")
eml_validate("ex.xml")
```

#### Upload the data package

Create data package and set identifiers

```
pkg <- new("DataPackage")

data_id <- generateIdentifier(mn, scheme = "uuid") # or set manually to your specific numbering scheme
metadata_id <- generateIdentifier(mn, scheme = "uuid")
```

Create a new metadata object and add it to the package

```
metadataObj <- new("DataObject",
                   id = metadata_id,
                   format ="eml://ecoinformatics.org/eml-2.1.1",
                   filename = "ex.xml")
                   
pkg <- addMember(pkg, metadataObj)

```

Create a data object and add it to the package

```
# Add our data file to the package
sourceObj <- new("DataObject",
                 id = data_id,
                 format = "text/csv",
                 filename = "files/my-data.csv")

pkg <- addMember(pkg, sourceObj, mo = metadataObj)
```

Add access to the OPC data admins group to all members of the package

```
pkg <- addAccessRule(pkg,
                    subject = "CN=opc-data-admins,DC=dataone,DC=org",
                    permission = c("read","write","changePermission"),
                    getIdentifiers(pkg))
```

Upload to the member node

```
packageId <- uploadDataPackage(d1c, pkg, public = TRUE)
```

## Update a dataset


Set packageId variable as the resource map of the package to update
(easily obtainable from website), and get the package.

```
packageId <- "resource_map_urn:uuid:74e6a16d-a971-4c8e-8b7b-fbf6ea9e1538"

pkg <- getDataPackage(d1c,
                      identifier = packageId,
                      lazyLoad = T,
                      limit = "0MB")
```

Set the path to the updated data file and the new custom identifier

```
updatedFile <- "new_file.csv"
updatedDataIdentifier <- generateIdentifier(d1c@mn, scheme = "UUID") # or set custom scheme
```

Select the file to replace by filename and replace it locally
```
oldFile <- selectMember(pkg, name = "sysmeta@fileName", value = "old_file.csv")
pkg <- replaceMember(pkg, oldFile, replacement = updatedFile, newId = updatedDataIdentifier)
```

Select the metadata file by extension (xml), read it into the R environment, and make some simple updates to associate the metadata with the new file information.

```
oldMetadata <- selectMember(pkg, name = "sysmeta@fileName", value = ".*xml")

doc <- read_eml(getObject(d1c@mn, oldMetadata))
doc$dataset$otherEntity$entityName <- updatedFile
doc$dataset$otherEntity$id <- updatedDataIdentifier
```

Write the new metadata file to disk and validate it

```
updatedMetadata <- "updated_metadata.xml"

write_eml(doc, updatedMetadata)
eml_validate(updatedMetadata) # should return TRUE
```

Create new metadata identifer, replace the old metadata file, and upload the updated package.

```
updatedMetadataIdentifier <- generateIdentifier(d1c@mn, scheme = "UUID") # or set custom scheme
pkg <- replaceMember(pkg, oldMetadata, replacement = updatedMetadata, newId = updatedMetadataIdentifier)

packageId <- uploadDataPackage(d1c, pkg, public = TRUE)
```

## Manually obsolete datasets

**Warning:** This should be done with great care to ensure that the identifiers used are correct, and that they are placed in the correct field in the system metadata. Doing this incorrectly can result in circular version chains, which are very problematic in Metacat.

I'm writing this as a function which you can load locally into your environment and use. It has some checks built in to try to prevent things from going wrong.

You can use this function to "merge" two version chains of a dataset if needed. In this case, the "obsoleted" pid would be the most recent version of the version chain you want to put on the bottom, and the "new" pid would be the *first* version of the version chain you want to put on the top.

Package 1: a1 - a2 - a3 - a4  
Package 2: b1 - b2

In the above example, to make b2 show up as the most recent version, you would set `obsoleted_pid` to a4, and `new_pid` to b1.

Note that this is often most useful to do on metadata pids, which is what metacatUI uses to display the "newer version" banner.

```
# obsolete_pid: the pid of the metadata object you want to obsolete
# new_pid: the pid of the object you want to obsolete the old version with

obsolete_object <- function(mn, obsolete_pid, new_pid){
    
    
    sys_obs <- dataone::getSystemMetadata(mn, obsolete_pid)
    sys_new <- dataone::getSystemMetadata(mn, new_pid)
    
    # Check that sysmeta fields to update are not already set
    if (!is.na(sys_obs@obsoletedBy)) {
        stop(message("pid: ", obsolete_pid, " already obsoleted by: ",
                     sys_obs@obsoletedBy, ". If you still wish to obsolete
                         this version chain please use the last PID in the version chain."))
    }
    if (!is.na(sys_new@obsoletes)) {
        stop(message("pid: ", new_pid, " already obsoletes: ",
                     sys_new@obsoletes, "."))
    }
    
    # update system metadata
    sys_obs@obsoletedBy <- new_pid
    sys_new@obsoletes   <- obsolete_pid
    
    updateSystemMetadata(mn, obsolete_pid, sys_obs)
    updateSystemMetadata(mn, new_pid, sys_new)
    
}
```

Set the new and obsoleted pids and run the function
```

new_pid <- "urn:uuid:7a27e8e0-dcf7-4271-87f1-f8d939e6dbd8"
obsolete_pid <- "urn:uuid:421401e2-12e5-4ca4-94e3-dcbff3ed8895_1"

obsolete_object(mn, obsolete_pid, new_pid)

```

## Archive objects

This can be done simply using the `archive` function from the `dataone` package.

I rarely use this in my workflow, and will often obsolete an orphaned package using the workflow above if I can.

```
pid_archive <- "pid"
archive(mn, pid_archive)
```

## Setting access policy on individual objects

The easiest way to do this is through modifying the system metadata of an object.

#### Add an access rule

```
objectId <- "some identifier"
sys <- getSystemMetadata(mn, objectId)
```

Create an access rules data frame using a handy function that allows you to create a data.frame
row-wise.

```
accessRules <- dplyr::tribble(~subject, ~permission,
                             "CN=opc-data-admins,DC=dataone,DC=org", "read",
                             "CN=opc-data-admins,DC=dataone,DC=org", "write",
                             "CN=opc-data-admins,DC=dataone,DC=org", "changePermission")


```
Add the access rule to the sysmeta and update.

```
sys <- addAccessRule(sys, accessRules)
updateSystemMetadata(mn, pid = objectId, sys)

```

#### Remove an access rule

Similar workflow to above, but uses `removeAccessRule`

```
objectId <- "some identifier"
sys <- getSystemMetadata(mn, objectId)

accessRules <- dplyr::tribble(~subject, ~permission,
                             "public", "read")


sys <- removeAccessRule(sys, accessRules)
updateSystemMetadata(mn, pid = objectId, sys)

```

## Copy a package as a template

Read in metadata to copy

```
metadataTemplateId <- "some identifier"

doc <- read_eml(getObject(mn, metadataTemplateId))
```

Remove `id` attribute from all of the entitites to allow for more flexibility in associating files with entities using metacatUI.

If there is only one entity (file) you can run this:
```
doc$dataset$otherEnity$id <- NULL 
```

If there is more than one entity, you need to run this:

```
doc$dataset$otherEntity <- lapply(doc$dataset$otherEntity, function(x){
    x$id <- NULL
    return(x)})
```

You can make other EML edits as needed.

Write the template file to disk.

```
write_eml(doc, "template_metadata.xml")
```

Upload the metadata file as you would in the [uploading data package section](#upload-the-data-package) 

```
pkg <- new("DataPackage")

metadata_id <- generateIdentifier(mn, scheme = "uuid")

metadataObj <- new("DataObject",
                   id = metadata_id,
                   format ="eml://ecoinformatics.org/eml-2.1.1",
                   filename = "template_metadata.xml")
                   
pkg <- addMember(pkg, metadataObj)

pkg <- addAccessRule(pkg,
                    subject = "CN=opc-data-admins,DC=dataone,DC=org",
                    permission = c("read","write","changePermission"),
                    getIdentifiers(pkg))

packageId <- uploadDataPackage(d1c, pkg, public = TRUE)

```

## Search datasets with Solr

More comprehensive information can be found [here](https://nceas.github.io/datateam-training/reference/solr-queries.html)

A list of queryable fields can be found [here](https://cn.dataone.org/cn/v2/query/solr)

The `dataone` query function allows for easy query of the Solr index. The query itself is composed of two primary parts:

* q: name:value pairs to search over, where the name comes from the the list of queryable fields linked above
* fl: result fields to return
* rows: the number of rows to return

This query will return the list of titles and identifiers where the word 'soil' is contained within the title, as a data frame.

```
result <- query(mn, list(q="title:*soil*",
                fl="title,identifier",
                rows="10"),
      as = "data.frame")
```

Here are a few example queries I use often. Solr is very powerful, so this is really just scratching the surface.

##### Return everything
```
result <- query(mn, list(q="*:*",
                               fl="*",
                               rows="20"),
                as = "data.frame")
```

##### Return objects a particular submitter published

```
result <- query(mn, list(q = 'submitter:"http://orcid.org/0000-0003-4703-1974"',
               fl = 'identifier,submitter,fileName, size',
               sort = 'dateUploaded+desc',
               rows='1000'),
      as = "data.frame")
```

##### Find objects by fileName

```
result <- query(mn, list(q = 'fileName:"my_file.csv"',
               fl = 'identifier,submitter,fileName, size',
               sort = 'dateUploaded+desc',
               rows='1000'),
      as = "data.frame")
```

##### Look for latest versions only

```
result <- query(mn, list(q = "submitter:*orcid.org/0000-000X-XXXX-XXXX* AND (*:* NOT obsoletedBy:*)",
                          fl = "identifier,rightsHolder,formatId",
                          start ="0",
                          rows = "1500"),
                     as="data.frame")
```
