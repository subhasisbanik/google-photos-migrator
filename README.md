# google-photos-migrator

This tool helps with the data downloaded from takeout.google.com and helps in fixing original date time of the media.

## Table of Contents

* [Background](#background)
* [Quick Start](#quick-start)
* [Structure of Google Takeout Export](#structure-of-google-takeout-export)
* [How to download Google Takeout content](#how-to-download-google-takeout-content)
* [What inputs do I need to provide?](#what-inputs-do-i-need-to-provide)
* [What does the tool do?](#what-does-the-tool-do)
* [Supported file types](#supported-file-types)
* [How are media files matched to JSON sidecar files?](#how-are-media-files-matched-to-json-sidecar-files)
* [Disclaimer?](#disclaimer)


## Background

Migration of Google Photos has been always an hassle especially with the original date time field embedded in the media. 

Google always does this where if you try to download all the data from [Google Takeout](https://takeout.google.com/), they will provide a zip dump of the whole dump.

I forked this tool to help me overcome some issues that I had when trying to make use of photos exported from Google Photos using [Google Takeout](https://takeout.google.com/).
 
Whilst it is great that I was able to use Google Takeout to extract all of my stored images from Google Photos at once, I found that some images were landing in the wrong place due to missing `DateTimeOriginal` EXIF timestamps. This timestamp determines as to how a image or media should be placed in library and this is generally messed up. 
In may case, I was planning my way out of Google Photos and moving my memories to One Drive simply because they are cheaper as of 2023.

This tool aims to eliminate some of those issues by reading the `photoTakenTime` timestamp from the JSON metadata files that are included in Google Takeout export and using it to:
- set a meaningful modification date on the file itself
- populate the `DateTimeOriginal` field in the EXIF metadata if this field is not already set 

### What is EXIF ?
EXIF (Exchangeable Image File Format) files store important data about photographs. Almost all digital cameras create these data files each time you snap a new picture. An EXIF file holds all the information about the image itself — such as the exposure level, where you took the photo, and any settings you used.

In this case, the EXIF data contains the original date and time of the image or media which gets modified when the data dump from Google Takeout is extracted.

## Quick Start

Example usage:

```
unzip -u -o <first.zip> -d ~/merged

yarn
yarn start --inputDir ~/merged --outputDir ~/output --errorDir ~/error
```


## Structure of Google Takeout export
Google Takeout provides you with one or more zip files, structured in a way that is fairly unintuitive and tricky to make use of directly.

Extracting the zip, you might find something similar to this: 

```
Extracted Takeout Zip:

  Google Photos
    2006-01-01
      IMG0305.jpg.json
    2020-09-01
      IMG1001.jpg
      IMG1001.jpg.json
    SomeAlbumName
      IMG1004.jpg
      IMG1004.jpg.json
      IMG1005.jpg
      IMG1005.json
  archive_browser.html
```

There are some interesting challenges to note here:

1. Each zip contains folders for certain dates and/or album names. These folders contain a mixture of image files and JSON metadata files. The JSON files include, amongst other things, a useful `photoTakenTime` property.
2. The date based folders don't always contain perfect pairs of images and JSON files, sometimes you get JSON files without a corresponding image. In the case that the export was split across multiple zips, it cannot be guaranteed that the corresponding JSON file will be placed in the same zip along with the actual file. So it is recommended to extract all zips and place in same directory. [Extract the zip(s)](#)
3. The naming convention for the JSON files seems inconsistent and has some interesting edge cases. For an image named `IMG123.jpg`, sometimes you get `IMG123.jpg.json` but sometimes it's just `IMG123.json` 
4. From what I can tell, the embedded metadata (e.g. EXIF / IPTC) in the image files is _not_ updated if changes are made within Google Photos, for example if the dates are updated using the Google Photos UI. Instead, Google's metadata comes out in the accompanying JSON files.
5. Whilst most of my images contained reasonable EXIF timestamps for the time they were taken (written by the phone's camera), a small number did not. My guess is that these images originated from other sources (e.g. they were shared with me or imported into the library by other means, and the source did not include a timestamp in the EXIF metadata)
6. Some file formats such as GIFs or MP4 videos don't have this metadata and thus also get sorted into the wrong place when run through tools that organise images based on the metadata timestamps.

## How to download Google Takeout content

The first step to using this tool is to request & download a `Google Takeout`. At the time of writing the steps to do this are:

1. Visit https://takeout.google.com
2. Deselect all products and then tick `Google Photos`
3. Click `All photo albums included`.
4. Keep all of the albums with year mentioned on it. **Do not select any other albums which Google creates because these are already present in the year based albums.**
5. **Important**: If you have custom albums (ones with non-date names), deselect these because the images will already be referenced by the date-based albums.
6. Click OK and move to the next step
7. Select "Export once"
8. Under "File type & size" it is recommended to increase the file size to 50GB. **Important**: If your collection is larger than this, Google will automatically created multiple zip with names ending with 001, 002 and onwards. 
9. Click "Create Export", wait for a link to be sent by email and then download the zip file. **This process takes substantial amount of time if you have some good amount of data, like I did**
10. Extract the zip file into a directory. The path of this directory will be what we pass into the tool as the `inputDir`.

## To extract and merge the zip(s), follow the steps below:
Run the following commands sequentially so that the multiple zips are stored in merged directory for further use by the tool:
```
unzip -u -o <first.zip> -d merged
unzip -u -o <second.zip> -d merged
```

**Important**: This step is very crucial as this will create the inputDir for the tool to consume.


## What inputs do I need to provide to the tool?

The tool takes in three parameters:

1. an `inputDir` directory path containing the extracted Google Takeout. (This will be available from the [Extracted folder](https://github.com/subhasisbanik/google-photos-migrator#to-extract-and-merge-the-zips-follow-the-steps-below))
2. an `outputDir` directory path where processed files will be moved to. This needs to be an empty directory and can be anywhere on the disk. 
3. an `errorDir` directory path where media with bad EXIF data that fail to process will be moved to. The folder can be empty. 
>Note: Though this folder contains media with bad EXIF data, still this is worked on by the tool and the output is churned to the Output folder.

The `inputDir` needs to be a single directory containing an _extracted_ zip from Google takeout. As described in the section above, it is important that the zip has been extracted into a directory (this tool doesn't extract zips for you) and that it is a single folder containing the whole Takeout (or if coming from multiple archives, that they have been properly merged together). 

For example:
```
Takeout
  Google Photos
    2017-04-03
    2020-01-01
    ...
```

## Configuring supported file types

In order to avoid touching files that are not photos or videos, this tool will only process files whose extensions are whitelisted in the configuration options. Any other files will be ignored and not included in the output.

To customise which files are processed, edit the `src/config.ts` file to suit your needs. For each extension you can also configure whether or not to attempt to read/write EXIF metadata for that file type.

The default configuration is as follows:
```
┌──────────┬─────────┐
│Extension │EXIF     │
├──────────┼─────────┤
│.jpeg     │true     │
│.jpg      │true     │
│.heic     │true     │
│.gif      │false    │
│.mp4      │true     │
│.png      │false    │
│.avi      │false    │
│.mov      │false    │
└──────────┴─────────┘
```

>Note: To be able to fix the date time issues of all the media, set true to all the above especially to jpeg, jpg, mp4 type data.

## What does the tool do?

The tool will do the following:
1. Find all "media files" with one of the supported extensions (see "Configuring supported file types" above) from the (nested) `inputDir` folder structure.
  
2. For each "media file":
   
   a. Look for a corresponding JSON metadata file and if found, read the `photoTakenTime` field
   
   b. Copy the media file to the output directory

   c. Update the file modification date to the `photoTakenTime` found in the JSON metadata
   
   d. If the file supports EXIF (e.g. JPEG images), read the EXIF metadata and write the `DateTimeOriginal` field if it does not already have a value in this field 

   e. If an error occurs whilst processing the file, copy it to the directory specified in the `errorDir` argument, so that it can be inspected manually or removed

3. Display a summary of work completed

## How are media files matched to JSON files?

The Google Takeout file/folder structure has some interesting inconsistencies/quirks which make it tricky to work with.

This tool makes a best-effort attempt to work with this structure. Read on to get a feel for how it works.

### Images have corresponding JSON files

Imagine you have an image named `foo.jpg`.

In the same directory, we also expect to find a JSON file containing metadata about the image. The pattern for how this is names seems inconsistent but in general, we seem to get either: `foo.json` or `foo.jpg.json`. This tool will look for either of those two files.

### Edited images (e.g. "foo-edited.jpg") don't have their own JSON files

It appears that images that were edited using Google Photos (e.g. rotated or edited on a mobile device) **don't** get their own JSON files.

For example, for image file `foo-edited.jpg` we _don't_ get `foo-edited.json` nor `foo-edited.jpg.json`. Instead we must rely on the JSON file from the original image.

To work around this, for any images with a suffix of `-edited`, this tool will ignore that suffix and look for the JSON file from the original image, e.g. `foo.json` or `foo.jpg.json`.

### Files with numbered suffixes (e.g. "foo(1).jpg") follow a different pattern for the naming of JSON files

I found that some of my images were named with a numeric suffix, such as `foo(1).jpg`.

Counter-intuitively, the corresponding JSON file for this doesn't follow the same pattern. Instead of `foo(1).json` or `foo(1).jpg.json`, it is instead named `foo.jpg(1).json`.

To support that, this tool will also check for files that have a number suffix in brackets and where that is the case, look for a JSON file with the pattern above.

## Disclaimer

This tool was forked from another Github repository. Even though the codebase if old, this still serves the purpose. All thanks to [Matt Wilson](https://github.com/mattwilson1024/google-photos-exif) for this amazing codebase and the detailed README. Most of the README is still same and the codebase needs some minor improvements due to deprecation of certain libraries which can be targetted later on.


I decided to make this public on GitHub because:
 - it was useful for me, so maybe it'll be useful for others in the future
 - future me might be thankful if I ever need to do this again


## Future Improvements
1. This tool has the possibility of working as a binary and run with the required configuration in Cloud
2. Fix the usage of deprecated libabries like @oclif/command
3. Change logging from verbose to info and debug
4. Some files like, for example Screenshot files do not have associated JSON files with them. This tool should be able to identify these files which are missing a JSON file, and update the date time with the name on the file since it appears the name also sometimes contain the correct date time on it.
