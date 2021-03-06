---
layout: post
title:  "Transferring Compressed Files from S3"
date:   2013-08-16
categories:
---

Requesting binary data over an `XMLHttpRequest` (XHR) is becoming more frequent.  Transferring binary data has benefits compared to sending a JSON representation when a native data format is desired or when a more compact data representation is needed compared to sending a string based format.  JSON tends to work fine when file sizes are small, but if large chunks of data need to be transferred to the client, a binary representation is more compact.

Amazon's S3 supports sending compressed data using gzip.  This feature is often used to serve compressed javascript or css files, reducing transfer times, and utilizing the browser's implementation of DEFLATE to decompress the files.  These features may also be used for arbitrary binary data, but there's a trick.

All data must be gzipped prior to uploading to S3.  For instance,

    gzip some-binary-file

will compress the file using gzip, substantially reducing the file size.  Uploading this file to S3 requires a few flags to be set.  If using `s3cmd`

    s3cmd put some-binary-file.gz s3://some-s3-bucket/ --mime-type "application/json" --add-header "Content-Encoding: gzip" --acl-public

The content encoding must be specified as `gzip`, and the mime-type *must be* `application/json`, even if the file is not JSON.

The binary XHR is written as,

    var xhr = new XMLHttpRequest();
    xhr.open('GET', url_to_binary_file);
    xhr.responseType = 'arraybuffer';
    
    // NOTE: Overriding the mime type from the client does not work!
    // xhr.overrideMimeType("application/json")
    
    xhr.onload = function() {
      console.log(xhr.response);
    }
    xhr.send();

Now the previously large file is transferred as a compressed file, but retrieved in it's original format!