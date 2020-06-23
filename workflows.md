# Common Processing Workflows

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

## Updating a package

update existing dataset
replace data files and metadata
create custom identifiers for all files and resource map

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
