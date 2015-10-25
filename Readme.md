# gulp-s3-upload
__Version 1.4.3__

Use for uploading assets to Amazon S3 servers.
This helps to make it an easy gulp task.

This package uses the [aws-sdk (node)](http://aws.amazon.com/sdk-for-node-js/).

[NPM](https://www.npmjs.com/package/gulp-s3-upload) / [Changelog](docs/changelog.md)

__See full details in the [Changelog](docs/changelog.md).__

## Install

    npm install gulp-s3-upload

## Usage

### Including + Setting Up Config

```js
    var gulp = require('gulp');
    var s3 = require('gulp-s3-upload')(config);
```

...where config is something like...

```js
var config = {
    accessKeyId: "YOURACCESSKEY",
    secretAccessKey: "YOUACCESSSECRET"
}

//  ...or...

var config = JSON.parse(fs.readFileSync('private/awsaccess.json'));

//  ...or to use IAM settings...

var config = { useIAM: true };

```

The optional `config` argument can include any option available (like `region`) available in the [AWS Config Constructor](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Config.html#constructor-property). By default all settings are undefined. 

**Per AWS best practices**, the recommended approach for loading credentials is to use the shared credentials file (`~/.aws/credentials`). You can also set the `aws_access_key_id` and `aws_secret_access_key` environment variables or specify values directly in the gulpfile via the `accessKeyId` and `secretAccessKey` options. If you have multiple profiles configured in your AWS credentials file, you can specify the profile name inline with the call to gulp.

If you are using **IAM** settings, just pass the noted config above in order to default to using IAM.  More information on using [IAM settings here](https://aws.amazon.com/documentation/iam/). 

```sh
AWS_PROFILE=myprofile gulp
```

You can also use a node_module like [config](https://www.npmjs.com/package/config) (+ [js-yaml](https://www.npmjs.com/package/js-yaml)) to load config files in your `gulpfile.js`.  You can also use `fs.readFileSync` to read from a local file to load your config.

Feel free to also include credentials straight into your `gulpfile.js`, though be careful about committing files with secret credentials in your projects!

Having AWS Key/Secrets may not be required by your AWS/IAM settings.  Errors thrown by the request should give your permission errors.


### Gulp Task

**NEW 1.4.0**: The s3 plugin now takes a second object parameter that exposes the [options hash for the AWS S3 Constructor Property](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#constructor-property).  Please note, if you have different configurations for different upload sets, you'll need to make a different task for each set.  You won't need the `accessKeyId` and secret since the plugin initially takes those in for the AWS Constructor.

Create a task.

```js
gulp.task("upload", function() {
    gulp.src("./dir/to/upload/**")
        .pipe(s3({
            Bucket: 'your-bucket-name', //  Required
            ACL:    'public-read'       //  Needs to be user-defined
        }, {
            // S3 Construcor Options, ie:
            maxRetries: 5
        }))
    ;
});
```


## Options

**Bucket (bucket)** *(required)*

Type: `string`

The bucket that the files will be uploaded to.

Other available options are the same as the ones found in the AWS-SDK docs for S3.  The end of the readme below for a list of availble AWS-SDK resources that this plugin constantly references.

**NOTE:** `Key`, `Body`, and `ContentType` are the only options availble in `putObject` that do **NOT** need to be defined because the gulp will handle these for you. If these are defined, the plugin will filter them out.

### gulp-s3-plugin options

### charset

Type: `string`

Use this to add a charset to the mimetype. `"charset=[CHARSET]"` gets appended to the mimetype if this is defined.

### etag_hash

Type: `string`

Default: `'md5'`

Use this to change the hashing of the files' ETags. The default is MD5.  More information on AWS's [Common Response Headers can be found here](http://docs.aws.amazon.com/AmazonS3/latest/API/RESTCommonResponseHeaders.html).  You shouldn't have to change this, but AWS says the "ETag may or may not be an MD5 diest of the object data", so this option has been implemented should any other case arise.


#### keyTransform (nameTransform)

Type: `function`

Use this to transform your file names before they're uploaded to your S3 bucket.
(Previously known as `name_transform`).

```js
    gulp.task("upload_transform", function() {
        gulp.src("./dir/to/upload/**")
            .pipe(s3({
                Bucket: 'example-bucket',
                ACL: 'public-read',
                keyTransform: function(relative_filename) {
                    var new_name = changeFileName(relative_filename);
                    // or do whatever you want
                    return new_name;
                }
            }))
        ;
    });
```

#### maps.ParamName {}

Type: `object` + `function`

**NEW IN 1.3**

Upon reviewing an issue with `metadataMap` and `manualContentEncoding`, a standard method for mapping each `s3.putObject` param was created. For now, `metadataMap` and `manualContentEncoding` are still available, but they will be depricated in the next major version (2.0).

Each property of the maps option must be a function and must match the paramter being mapped. The files' `keyname` will be passed through (keep in mind, this is after any `keyTransform` calls).  The function should return the output S3 expects. [You can find more information and the available options here](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property).

For example, to define metadataMap and separate expirations in this way:

```js
    var metadata_collection = { /* your info here */ };
    var expirations = { /* your info here */ };

    gulp.task("upload", function() {
        gulp.src("./dir/to/upload/**")
        .pipe(s3({
            Bucket: 'example-bucket',
            ACL: 'public-read',
            maps: {
                Metadata: function(keyname) {
                    path.basename(keyname); // just get the filename
                    return metadata_collection[keyname]; // return an object
                },
                Expires: function(keyname) {
                     path.basename(keyname); // just get the filename
                     return new Date(expirations[keyname]);
                }
            }
        }));
    });
```

If anything but a function is passed through, nothing will happen. If you want to send a consistent value to all of your files this way, just simply set the option straight in the main options like so:

```js
    var expires = new Date();
    expires.setUTCFullYear(2020);

    gulp.task("upload", function() {
        gulp.src("./dir/to/upload/**")
        .pipe(s3({
            Bucket: 'example-bucket',
            ACL: 'public-read',
            Metadata: {
                "example1": "This is an example"
            },
            Expires: expires
        }));
    });
```


#### metadataMap

**NOTE**: It is preferred you use the maps.ParamsName method to define and map specific metadata to files.  Also, if you set both `maps.Metadata` and this, `metadataMap` will take precedence. 

Type: `object` or `function`

If you have constant metadata you want to attach to each object,
just define the object, and it will be included to each object.
If you wish to change it per object, you can pass a function through
to modify the metadata based on the (transformed) keyname.

Example (passing an `object`):
```js
    gulp.task("upload", function() {
        gulp.src("./dir/to/upload/**")
        .pipe(s3({
            Bucket: 'example-bucket',
            ACL: 'public-read',
            metadataMap: {
                "uploadedVia": "gulp-s3-upload",
                "exampleFlag":  "Asset Flag"
            }
        }));
    });
```
Passing the `s3.putObject` param option `Metadata` is effectively the same thing
as passing an `object` to `metadataMap`.  `Metadata` is defined and `metadataMap` is not
it will use the object passed to `Metadata` as metadata for all the files that
will be uploaded.  If both `Metadata` and `metadataMap` are defined, `Metadata` will take
precedence and be added to each file being uploaded.

Example (passing a `function`):

```js
    // ... setup gulp-s3-upload ...
    var path = require('path');
    var metadata_collection = {
        "file1.txt": {
            "uploadedVia": "gulp-s3-upload",
            "example": "Example Data"
        },
        "file2.html": {
            "uploadedVia": "gulp-s3-upload"
        }
    };

    gulp.task("uploadWithMeta", function() {
        gulp.src("./upload/**")
        .pipe(s3({
            Bucket: 'example-bucket',
            ACL: 'public-read',
            metadataMap: function(keyname) {
                path.basename(keyname); // just get the filename
                return metadata_collection[keyname]; // return an object
            }
        }));
    });
```

When passing a function, it's important to note that the file
will already be transformed either by the `keyTransform` you defined
or by the default function which creates a keyname relative to
your S3 bucket, e.g. you can get "example.txt" or "docs/example.txt"
depending on how it was structured locally (hence why in the example,
the `path` module is used to just get the filename).

**Note:** You should be responsible for handling mismatching/unmatched keynames
to the metadata you're mapping.


#### mimeTypeLookup

Type: `function`

Use this to transform what the key that is used to match the MIME type when uploading to S3.

```js
    gulp.task("upload", function() {
        gulp.src("./dir/to/upload/**")
        .pipe(s3({
            Bucket: 'example-bucket',
            ACL: 'public-read',
            mimeTypelookup: function(original_keyname) {
                return original_keyname.replace('.gz', ''); // ignore gzip extension
            },
        }));
    });
```


#### uploadNewFilesOnly

Type: `boolean`

Set `uploadNewFilesOnly: true` if you only want to upload new files and not
overwrite existing ones.


#### manualContentEncoding

**NOTE**: It is preferred you use the maps.ParamsName method to define and map specific Content Encoding values to files.  If you set both `maps.ContentEncoding` and `manualContentEncoding`, `manualContentEncoding` will take priority.

Type: `string` or `function`

If you want to add a custom content-encoding header on a per file basis, you can
define a function that determines the content encoding based on the keyname.
Defining a `string` is like passing the `s3.putObject` param option `ContentEncoding`.

Example (passing a `string`):
```js
    gulp.task("upload", function() {
        gulp.src("./dir/to/upload/**")
        .pipe(s3({
            Bucket: 'example-bucket',
            ACL: 'public-read',
            manualContentEncoding: 'gzip'
        }));
    });
```

Example (passing a `function`):
```js
    gulp.task("upload", function() {
        gulp.src("./dir/to/upload/**")
        .pipe(s3({
            Bucket: 'example-bucket',
            ACL: 'public-read',
            manualContentEncoding: function(keyname) {
                var contentEncoding = null;

                if (keyname.indexOf('.gz') !== -1) {
                  contentEncoding = 'gzip';
                }
                return contentEncoding;
            }
        }));
    });
```

## AWS-SDK References

* [AWS Config Constructor](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Config.html#constructor-property)
* [Configuring the AWS Node.js SDK](http://docs.aws.amazon.com/AWSJavaScriptSDK/guide/node-configuring.html)
* [S3 putObject](http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#putObject-property)
* [Access Control List (ACL) Overview](http://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html)

----------------------------------------------------

## License

Copyright (c) 2015, [Caroline Amaba](mailto:github@carolineamaba.com)

Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted, provided that the above copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
