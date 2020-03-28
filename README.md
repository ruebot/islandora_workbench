# Islandora Workbench

A command-line tool that allows creation, updating, and deletion of Islandora content. Islandora Workbench communicates with Islandora via REST, so it can be run anywhere - it does not need to run on the Islandora server.

Islandora Workbench started as a Python port of https://github.com/mjordan/claw_rest_ingester, but has a lot of additional functionality.

## Requirements

* Python 3.2 or higher
   * The [ruamel.yaml](https://yaml.readthedocs.io/en/latest/index.html) library
   * The [Requests](https://2.python-requests.org/en/master/) library
* An [Islandora 8](https://islandora.ca/) repository
   * The JSON:API module is not enabled by default. You must enable it manually.
   * Drupal's REST API must have "basic" authentication enabled (it is on by default for JSON:API)
   * You must enable the following two REST Resources at `admin/config/services/rest`:
     * Field
     * Field storage
        * For these two resources, set "Granularity" to "Method" and check "GET", "Accepted request formats" to "JSON", and "Authentication providers" to "basic_auth".


## Installation

* `git clone https://github.com/mjordan/islandora_workbench.git`

If you don't already have the two required libraries installed, clone this repo as above, and then use `setup.py`:

* `sudo python3 setup.py install`

## Usage

`./workbench --config config.yml`

where `--config` is the path to a YAML file like this:

```yaml
task: create
host: "http://localhost:8000"
username: admin
password: islandora
content_type: islandora_object
input_dir: input_data
input_csv: metadata.csv
media_use_tid: 16
drupal_filesystem: "fedora://"
id_field: id
```

* `task` is one of 'create', 'update', delete', or 'add_media'.
* `host` is the hostname, including port number if not 80, of your Islandora repository.
* `username` is the username used to authenticate the requests.
* `password` is the user's password.
* `content_type` is the machine name of the Drupal node content type you are creating or updating.
* `input_dir` is the full or relative path to the directory containing the files and metadata CSV file.
* `input_csv` is the filename of the CSV metadata file, which must be in the directory named in '--input_dir'.
* `media_use_tid` is the term ID for the Media Use term you want to apply to the media.
* `media_type` (singular) specifies whether the media being created in the 'create' or 'add_media' task is an `image`, `file`, `document`, `audio`, or `video` (or other media type that exists in the target Islandora).
* `media_types` (plural) provides a mapping bewteen file extensions and media types. Note: either `media_type` or `media_types` is required. More detail provided in the "Setting Media Types" section below.
* `drupal_filesystem` is either 'fedora://' or 'public://'.
* `delimiter` is the delimiter used in the CSV file, for example, "," or "\t". If omitted, defaults to ",".
* `id_field` is the name of the field in the CSV that uniquely identifies each record. If omitted, defaults to 'id'.
* `published` determines if nodes are published or not. Applies to 'create' task only. Defaults to `True`; set to `False` if you want the nodes to be unpublished. Note that whether or not a node is published can also be set at a node level in the CSV file in the `status` base field, as described in the "Base Fields" section below. Values in the CSV override the value of `published` set here.
* `validate_title_length`: Whether or not to check if `title` values in the CSV exceed Drupal's maximum allowed length of 255 characters. Defaults to `True`. Set to `False` if you are using a module that lets you override Drupal's maximum title length, such as [Node Title Length](https://www.drupal.org/project/title_length) or [Entity Title Length](https://www.drupal.org/project/entity_title_length).
* `pause` defines the number of seconds to pause between each REST request to Drupal. Include it in your configuration to lessen the impact of Islandora Workbench on your site during large jobs, for example `pause: 1.5`.

All configuration settings are required for the "create" task if its entry in the list above does not specify a default value. The "update", "delete", and "add_media" tasks do not require all of the options, as illustrated below. Optional configuration settings are described in the sections below where they apply.

## Checking configuration and input data

You should always (always, [I can't stress that enough](https://www.youtube.com/watch?v=2ZgtEdFAg3s)) check your configuration and input prior to creating, updating, or deleting content. You can do this by running Workbench with the `--check` option, e.g.:

`./workbench --config config.yml --check`

If you do this, Workbench will check the following and report any errors that require your attention before proceding:

* Whether your configuration file contains all required values.
* Whether the `host` you provided will accept the `username` and `password` you provided.
* Whether your CSV file contains required columns headers, including the field defined as the unique ID for each record (defaults to "id" if the `id_field` key is not in your config file)
* Whether your CSV column headers correspond to existing Drupal field machine names.
* Whether all Drupal fields that are configured to be required are present in the CSV file.
* Whether the files named in the CSV file are present (but this check is skipped if `allow_missing_files: True` is present in your config file for "create" tasks).
* If the `langcode` field is present in your CSV, whether values in it are valid Drupal language codes.
* Whether values in the `title` field exceed Drupal's maximum length for titles of 255 characters (but this check is skipped if `validate_title_length` is set to `False`).
* Whether either `media_type` or `media_types` is present in your configuration file.
* Whether each row contains the same number of columns as there are column headers.

## Creating nodes from the sample data

Using the sample data and configuration file, the output of `./workbench --config create.yml` should look something like:

```
Node for 'Small boats in Havana Harbour' created at http://localhost:8000/node/52.
+File media for IMG_1410.tif created.
Node for 'Manhatten Island' created at http://localhost:8000/node/53.
+File media for IMG_2549.jp2 created.
Node for 'Looking across Burrard Inlet' created at http://localhost:8000/node/54.
+Image media for IMG_2940.JPG created.
Node for 'Amsterdam waterfront' created at http://localhost:8000/node/55.
+Image media for IMG_2958.JPG created.
Node for 'Alcatraz Island' created at http://localhost:8000/node/56.
+Image media for IMG_5083.JPG created.
```

## Using your own input data

### The files

The directory that contains the data to be ingested (identified by the `input_dir` config option) needs to contain a CSV file with field content and any accompanying media files you want to add to the newly created nodes:

```
your_folder/
├── image1.JPG
├── pic_saturday.jpg
├── image-27262.jpg
├── IMG_2958.JPG
├── someimage.jpg
└── metadata.csv
```

The names of the image/PDF/video/etc. files can take any form you want since they are included in the `file` column of the CSV file. Files of any extension are allowed.

By defualt, if the `file` value for a row is empty, Workbench's `--check` option will show an error. But, in some cases you may want to create a node but not add any media. If you add `allow_missing_files: True` to your config file for "create" tasks, you can leave the `file` cell in your CSV for that item empty.

### The CSV file

Metadata that is to be added to new or existing nodes is contained in the CSV file. Field values do not need to be wrapped in double quotation marks (`"`), unless they contain an instance of the delimiter character. Field values are either strings (for string or text fields), integers (for `field_weight`, for example), `1` or `0` for binary fields, or IDs (for taxonomy terms or collections).

Single-valued and multi-valued fields of the following types can be added:

* base fields
* text (plain, plain long, etc.) fields
* integer fields
* boolean fields, with values 1 or 0
* EDTF date fields
* entity reference (taxonomy and linked node) fields
* typed relation (taxonomy and linked node) fields

#### Required fields

* For the `create` task, `title`, `id` (or whatever field is identified in the `id_field` configuration option), and `file` are required. Empty values in the `file` field are allowed, in which case a node will be created but it will have no attached media.
* For the `update`, `delete`, and `add_media` tasks, the `node_id` field is required.
* For the `add_media` task, `file` is required, but for this task, `file` must contain a filename.
 
#### Base fields

Base fields are basic node properties, shared by all content types. The base fields you can include in your CSV file are:

* `title`: This field is required for all rows in your CSV for the `create` task. Optional for the 'update' task. Drupal limits the title's length to 255 characters, and Workbench will check that titles are less than 255 characters unless your configuration file contains `validate_title_length: False` as described above.
* `promote`: Promoted to front page. Optional. If included, use `1` (promoted) or `0` (not promoted) as values. If absent, is set to the default value for your content type.
* `status`: Whether the node is published. Optional. If included, use `1` (published) or `0` (unpublished) as values. If absent, is set to the default value for your content type.
* `sticky`: Sticky at top of lists. Optional. If included, use `1` (sticky) or `0` (not sticky) as values. If absent, is set to the default value for your content type.
* `langcode`: The language of the node. Optional. If included, use one of Drupal's language codes as values (common values are 'en', 'fr', and 'es'; the entire list can be seen [here](https://git.drupalcode.org/project/drupal/-/blob/8.8.x/core/lib/Drupal/Core/Language/LanguageManager.php#L224). If absent, Drupal sets the value to the default value for your content type.

#### Single-valued fields

You can include additional fields that will be added to the nodes. The column headings in the CSV file must match machine names of fields that exist in the target Islandora content type.

For example, using the fields defined by the Islandora Defaults module for the "Repository Item" content type, your CSV file could look like this:

```csv
file,title,id,field_model,field_description,field_rights,field_extent,field_access_terms,field_member_of
myfile.jpg,My nice image,obj_00001,24,"A fine image, yes?",Do whatever you want with it.,There's only one image.,27,45
```

In this example, the term ID for the tag you want to assign in `field_access_terms` is 27, and the node ID of the collection you want to add the object to (in `field_member_of`) is 45.

#### Multivalued fields

For multivalued fields, separate the values within a field with a pipe (`|`), like this:

```
file,title,field_my_multivalued_field
IMG_1410.tif,Small boats in Havana Harbour,foo|bar
IMG_2549.jp2,Manhatten Island,bif|bop|burp
```

This works for string fields as well as reference fields, e.g.:

```
file,title,field_my_multivalued_taxonomy_field
IMG_1410.tif,Small boats in Havana Harbour,35|46
IMG_2549.jp2,Manhatten Island,34|56|28
```

Drupal strictly enforces the maximum number of values allowed in a field. If the number of values in your CSV file for a field exceed a field's configured maximum number of fields, Workbench will only populate the field to the field's configured limit.

The subdelimiter character defaults to a pipe (`|`) but can be set in your config file using the `subdelimiter: ";"` option.

#### Typed Relation fields

Unlike most field types, which take a string or an integer as their value in the CSV file, fields that have the "Typed Relation" type take structured values that need to be entered in a specific way in the CSV file. An example of this type of field is the "Linked Agent" field in the Repository Item content type created by the Islandora Defaults module.

The structure of values for this field encode a namespace (indicating the vocabulary the relation is from), a relation type, and a target ID (which identifies what the relation refers to, such as a specific taxonomy term), each separated by a colon (`:`). The first two parts, the namespace and the relation type, come from the "Available Relations" section of the field's configuration, which looks like this (using the "Linked Agent" field's configuration as an exmple):

![Relations example](docs/images/relators.png)

In the node edit form, this structure is represented as a select list of the types (the namespace is not shown) and, below that, an autocomplete field to indicate the relation target, e.g.:

![Linked agent example](docs/images/linked_agent.png)

To include these kind of values in a CSV field, we need to use a structured string as described above (namespace:relationtype:targetid). For example:

`relators:art:30`

> Note that the structure required for typed relation values in the CSV file is not the same as the structure of the relations configuration depicted in the first screenshot above; the CSV values use only colons to seprate the three parts, but the field configuration uses a colon and then a pipe (|) to structure its values.

In this example of a CSV value, `relators` is the namespace that the relation type `art` is from (the Library of Congress [Relators](http://id.loc.gov/vocabulary/relators.html) vocabulary), and the target taxonomy term ID is `30`. In the screenshot above showing the "Linked Agent" field of a node, the value of the Relationship Type select list is "Artist (art)", and the value of the associated taxonomy term field is the person's name that has the taxonomy term ID "30" (in this case, "Jordan, Mark"):

If you want to include multiple typed relation values in a single field of your CSV file (such as in "field_linked_agent"), separate the three-part values with the same subdelimiter character you use in other fields, e.g. (`|`) (or whatever you have configured as your `subdelimiter`):

`relators:art:30|relators:art:45`

## Setting media types

The media type for a given file (for example, `image`, `file`, `document`, `audio`, or `video`) can be set in two ways in Workbench's configuration for `create` and `add_media` tasks. One of the following two configuration options is required.

1. Globally, via the `media_type` configuration option. If this is present (for example `media_type: document`), all media created by Workbench will be assigned that media type. Use this option if all of the files in your batch are to be assigned the same media type.
1. On a per-file basis, via a mapping from file extensions to media types. This is done by including a mapping in the `media_types` option (notice the plural) in your configuration file like this one:

   ```
   media_types:
    - file: ['tif', 'tiff', 'jp2', 'zip', 'tar']
    - document: ['pdf', 'doc', 'docx', 'ppt', 'pptx']
    - image: ['png', 'gif', 'jpg', 'jpeg']
    - audio: ['mp3', 'wav', 'aac']
    - video: ['mp4']
    - extracted_text: ['txt']
   ```
   Use this option if the files in your batch are not to be assigned the same media type. If a file's extension is not in one of the extension lists, the media is assigned the `file` type.

If both `media_type` and `media_types` are included in the config file, the mapping is ignored and the media type assigned in `media_type` is used.

## Updating nodes

You can update nodes by providing a CSV file with a `node_id` column plus field data you want to update. Updates preserve any values in the fields, they don't replace the values. The other column headings in the CSV file must match machine names of fields that exist in the target Islandora content type. Currently, text fields, taxonomy fields, linked node fields (e.g. "Member of" for collection nodes), and typed relation fields can be updated.

For example, using the fields defined by the Islandora Defaults module for the "Repository Item" content type, your CSV file could look like this:

```csv
node_id,field_description,field_rights,field_access_terms,field_member_of
100,This is my new title,I have changed my mind. This item is yours to keep.,27,45
```

Multivalued fields are also supported in the update task. See details in the "Multivalued fields" section above.

The config file for update operations looks like this (note the `task` option is 'update'):

```yaml
task: update
host: "http://localhost:8000"
username: admin
content_type: islandora_object
password: islandora
input_dir: input_data
input_csv: update.csv
```

## Deleting nodes

You can delete nodes by providing a CSV file that contains a single column, `node_id`, like this:

```csv
node_id
95
96
200
```

The config file for update operations looks like this (note the `task` option is 'delete'):

```yaml
task: delete
host: "http://localhost:8000"
username: admin
password: islandora
input_dir: input_data
input_csv: delete.csv
```

## Adding media to nodes

You can add media to nodes by providing a CSV file with a `node_id` column plus a `file` field that contains the name of the file you want to add. For example, your CSV file could look like this:

```csv
node_id,file
100,test.txt
```

The config file for update operations looks like this (note the `task` option is 'add_media'):

```yaml
task: add_media
host: "http://localhost:8000"
username: admin
password: islandora
input_dir: input_data
input_csv: add_media.csv
media_use_tid: 14
drupal_filesystem: "fedora://"
```

## Logging

Islandora Workbench writes a log file for all tasks to `workbench.log` in the workbench directory, unless you specify an alternative log file location using the `log_file_path` configuration option, e.g.:

`log_file_path: /tmp/mylogfilepath.log`

 By default, new entries are appended to this log, unless you indicate that the log file should be overwritten each time Workbench is run by providing the `log_file_mode` configuration option with a value of "w":

 `log_file_mode: w`

## Contributing

Bug reports, improvements, feature requests, and PRs welcome. Before you open a pull request, please open an issue.

If you open a PR, please check your code with pycodestyle:

`pycodestyle --show-source --show-pep8 --max-line-length=200 .`

Also provide tests where applicable. Sample tests are available in the `tests` directory. Note that these tests query a live Islandora instance, so you should write them assuming there is one running at localhost:8000. Run tests using the following:

`python3 -m unittest tests/*.py`

## License

The Unlicense.
