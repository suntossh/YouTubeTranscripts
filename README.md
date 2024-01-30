
# YouTube Transcript/Subtitle API (including automatically generated subtitles and subtitle translations)  


This is a python API which allows you to get the transcript/subtitles for a given YouTube video. It also works for automatically generated subtitles, supports translating subtitles and it does not require a headless browser, like other selenium based solutions do!

## Install

It is recommended to [install this module by using pip](https://pypi.org/project/youtube-transcript-api/):

```
pip install youtube-transcript-api
```

If you want to use it from source, you'll have to install the dependencies manually:

```
pip install -r requirements.txt
```

You can either integrate this module [into an existing application](#api), or just use it via an [CLI](#cli).

## API

The easiest way to get a transcript for a given video is to execute:

```python
from youtube_transcript_api import YouTubeTranscriptApi

YouTubeTranscriptApi.get_transcript(video_id)
```

> **Note:** By default, this will try to access the English transcript of the video. If your video has a different language, or you are interested in fetching a different language's transcript, please read the section below.

This will return a list of dictionaries looking somewhat like this:

```python
[
    {
        'text': 'Hey there',
        'start': 7.58,
        'duration': 6.13
    },
    {
        'text': 'how are you',
        'start': 14.08,
        'duration': 7.58
    },
    # ...
]
```
### Retrieve different languages

You can add the `languages` param if you want to make sure the transcripts are retrieved in your desired language (it defaults to english).

```python
YouTubeTranscriptApi.get_transcript(video_id, languages=['de', 'en'])
```

It's a list of language codes in a descending priority. In this example it will first try to fetch the german transcript (`'de'`) and then fetch the english transcript (`'en'`) if it fails to do so. If you want to find out which languages are available first, [have a look at `list_transcripts()`](#list-available-transcripts).

If you only want one language, you still need to format the `languages` argument as a list

```python
YouTubeTranscriptApi.get_transcript(video_id, languages=['de'])
```
### Batch fetching of transcripts

To get transcripts for a list of video ids you can call:

```python
YouTubeTranscriptApi.get_transcripts(["video_id1", "video_id2"], languages=['de', 'en'])
```

`languages` also is optional here.

### Preserve formatting

You can also add `preserve_formatting=True` if you'd like to keep HTML formatting elements such as `<i>` (italics) and `<b>` (bold).

```python
YouTubeTranscriptApi.get_transcripts(video_ids, languages=['de', 'en'], preserve_formatting=True)
```

### List available transcripts

If you want to list all transcripts which are available for a given video you can call:

```python
transcript_list = YouTubeTranscriptApi.list_transcripts(video_id)
```

This will return a `TranscriptList` object  which is iterable and provides methods to filter the list of transcripts for specific languages and types, like:

```python
transcript = transcript_list.find_transcript(['de', 'en'])
```

By default this module always picks manually created transcripts over automatically created ones, if a transcript in the requested language is available both manually created and generated. The `TranscriptList` allows you to bypass this default behaviour by searching for specific transcript types:

```python
# filter for manually created transcripts
transcript = transcript_list.find_manually_created_transcript(['de', 'en'])

# or automatically generated ones
transcript = transcript_list.find_generated_transcript(['de', 'en'])
```

The methods `find_generated_transcript`, `find_manually_created_transcript`, `find_transcript` return `Transcript` objects. They contain metadata regarding the transcript:

```python
print(
    transcript.video_id,
    transcript.language,
    transcript.language_code,
    # whether it has been manually created or generated by YouTube
    transcript.is_generated,
    # whether this transcript can be translated or not
    transcript.is_translatable,
    # a list of languages the transcript can be translated to
    transcript.translation_languages,
)
```

and provide the method, which allows you to fetch the actual transcript data:

```python
transcript.fetch()
```

### Translate transcript

YouTube has a feature which allows you to automatically translate subtitles. This module also makes it possible to access this feature. To do so `Transcript` objects provide a `translate()` method, which returns a new translated `Transcript` object:

```python
transcript = transcript_list.find_transcript(['en'])
translated_transcript = transcript.translate('de')
print(translated_transcript.fetch())
```

### By example
```python
from youtube_transcript_api import YouTubeTranscriptApi

# retrieve the available transcripts
transcript_list = YouTubeTranscriptApi.list_transcripts('video_id')

# iterate over all available transcripts
for transcript in transcript_list:

    # the Transcript object provides metadata properties
    print(
        transcript.video_id,
        transcript.language,
        transcript.language_code,
        # whether it has been manually created or generated by YouTube
        transcript.is_generated,
        # whether this transcript can be translated or not
        transcript.is_translatable,
        # a list of languages the transcript can be translated to
        transcript.translation_languages,
    )

    # fetch the actual transcript data
    print(transcript.fetch())

    # translating the transcript will return another transcript object
    print(transcript.translate('en').fetch())

# you can also directly filter for the language you are looking for, using the transcript list
transcript = transcript_list.find_transcript(['de', 'en'])  

# or just filter for manually created transcripts  
transcript = transcript_list.find_manually_created_transcript(['de', 'en'])  

# or automatically generated ones  
transcript = transcript_list.find_generated_transcript(['de', 'en'])
```

### Using Formatters
Formatters are meant to be an additional layer of processing of the transcript you pass it. The goal is to convert the transcript from its Python data type into a consistent string of a given "format". Such as a basic text (`.txt`) or even formats that have a defined specification such as JSON (`.json`), WebVTT (`.vtt`), SRT (`.srt`), Comma-separated format (`.csv`), etc...

The `formatters` submodule provides a few basic formatters to wrap around you transcript data in cases where you might want to do something such as output a specific format then write that format to a file. Maybe to backup/store and run another script against at a later time.

We provided a few subclasses of formatters to use:

- JSONFormatter
- PrettyPrintFormatter
- TextFormatter
- WebVTTFormatter
- SRTFormatter

Here is how to import from the `formatters` module.

```python
# the base class to inherit from when creating your own formatter.
from youtube_transcript_api.formatters import Formatter

# some provided subclasses, each outputs a different string format.
from youtube_transcript_api.formatters import JSONFormatter
from youtube_transcript_api.formatters import TextFormatter
from youtube_transcript_api.formatters import WebVTTFormatter
from youtube_transcript_api.formatters import SRTFormatter
```

### Provided Formatter Example
Lets say we wanted to retrieve a transcript and write that transcript as a JSON file in the same format as the API returned it as. That would look something like this:

```python
# your_custom_script.py

from youtube_transcript_api import YouTubeTranscriptApi
from youtube_transcript_api.formatters import JSONFormatter

# Must be a single transcript.
transcript = YouTubeTranscriptApi.get_transcript(video_id)

formatter = JSONFormatter()

# .format_transcript(transcript) turns the transcript into a JSON string.
json_formatted = formatter.format_transcript(transcript)


# Now we can write it out to a file.
with open('your_filename.json', 'w', encoding='utf-8') as json_file:
    json_file.write(json_formatted)

# Now should have a new JSON file that you can easily read back into Python.
```

**Passing extra keyword arguments**

Since JSONFormatter leverages `json.dumps()` you can also forward keyword arguments into `.format_transcript(transcript)` such as making your file output prettier by forwarding the `indent=2` keyword argument.

```python
json_formatted = JSONFormatter().format_transcript(transcript, indent=2)
```

### Custom Formatter Example
You can implement your own formatter class. Just inherit from the `Formatter` base class and ensure you implement the `format_transcript(self, transcript, **kwargs)` and `format_transcripts(self, transcripts, **kwargs)` methods which should ultimately return a string when called on your formatter instance.

```python

class MyCustomFormatter(Formatter):
    def format_transcript(self, transcript, **kwargs):
        # Do your custom work in here, but return a string.
        return 'your processed output data as a string.'

    def format_transcripts(self, transcripts, **kwargs):
        # Do your custom work in here to format a list of transcripts, but return a string.
        return 'your processed output data as a string.'
```


