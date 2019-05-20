# Overview

Extension to basic preprocessor class with useful methods and functions.

# Usage

Usually when we create preprocessors we inherit from `BasePreprocessor`:

```python

class Preprocessor(BasePreprocessor):
  ...
```

To use all features of preprocessor_ext we should inherit from `BasePreprocessorExt` instead:

```python

class Preprocessor(BasePreprocessorExt):
  ...
```

# Features

## Simplified tag processing workflow

Usually to process tags in Markdown-files we write something like this:

```python
class Preprocessor(BasePreprocessor):
  ...
  def _process_tags(content):
    def sub(match):
      # process the tag
      return processed_string

    self.pattern.sub(sub, content)

  def apply(self):
    self.logger.info('Applying preprocessor')
    for markdown_file_path in self.working_dir.rglob('*.md'):
        with open(markdown_file_path, encoding='utf8') as markdown_file:
            content = markdown_file.read()

        processed_content = self._process_tags(content)

        if processed_content:
            with open(markdown_file_path, 'w') as markdown_file:
                markdown_file.write(processed_content)
        self.logger.info('Preprocessor applied')
```

BasePreprocessorExt saves us from writing these many lines of the code which appear unchanged in most preprocessors.

So now instead of all that code we will write:

```python
class Preprocessor(BasePreprocessorExt):
  ...
  def _process_tag(match):
    # process the tag
    return processed_string

  def apply(self):
    self._process_tags_for_all_files(func=self._process_tag)
        self.logger.info('Preprocessor applied')
```

As a bonus when using this workflow we get additional capabilities of logging and outputting warnings.

## Issuing warnings

When using `_process_tags_for_all_files` method to process tags we can also take advantage of `_warning` method.

What this method does:
1. Prints warning message to user and adds the md-file name to this message.
2. Logs this message.
3. May also add to logged message context of the tag where the problem occured.
4. May also add to logged message the error traceback.
5. If debug=true, #3 and #4 are also added to the message which is printed to user.

Simple example:

```python
class Preprocessor(BasePreprocessorExt):
  ...
  def _process_tag(match):
    try:
      config = open(self.options.get('config'))
    except FileNotFoundError as e:
      self._warning('Config file not found! Using default', error=e)
    ...
```

Here if the exception gets triggered, user will see something like this in console:

```bash
Parsing config
Done

Applying preprocessor my_preprocessor
WARNING: [index.md] Config file not found! Using default

Done

Applying preprocessor _unescape
Done

────────────────────
Result: slug.pre

```

As we see, we've only supplied the message, but the preprocessor also added the `WARNING:` prefix and current file name to console. If we'd run the make command in debug mode (with `-d --debug` flag), we would also see full traceback of the error. In any case, traceback is stored in log.

## Getting tag context

Sometimes we want to show the context of the tag, where we met some problems. By context I mean some words before the tag, some contents of the tag body and some words after the tag. It's really useful for debugging large md-files, with context you usually can identify the place in the document which causes errors.

For this purpose `BasePreprocessorExt` class has a handy method called `get_tag_context`. Give it the match object you are currently working with and it will return you the string with tag context.

For example:

```python
class Preprocessor(BasePreprocessorExt):
  ...
  def _process_tag(match):
    try:
      config = open(self.options.get('config'))
    except FileNotFoundError as e:
      context = self.get_tag_context(match)
      print(f'Config not found, check the tag:\n{context}')
    ...
```

This will print:

```bash
Parsing config
Done

Applying preprocessor my_preprocessor

Config not found, check the tag:
...amet, consectetur adipisicing elit. Dolores ipsum non nisi voluptatum alias.

<my_tag param="value" config="wrong/path">
    Tag body consectetur adipisicing elit. Voluptatem.
</template>

End of document.
```

Now user can easily understand where's the problem in his document.

`get_tag_context` function accepts two parameters:

`limit` (default: `100`) — number of characters included in context from before the tag, after the tag, and of the tag body;
`full_tag` (default: `False`) — if this is True, the tag body is copied into context without cropping (useful for relatively small expected tag bodies).

## Sending context to warning

One last thing to use the full power of `BasePreprocessorExt` warnings:

`self._warning` accepts the context parameter. You can send there the context string. The context is only shown to user in the console if the debug mode is on, but it is always saved in the log file.

Example:

```python
class Preprocessor(BasePreprocessorExt):
  ...
  def _process_tag(match):
    try:
      config = open(self.options.get('config'))
    except FileNotFoundError as e:
      self._warning('Config file not found! Using default',
                    context=self.get_context(match),
                    error=e)
    ...
```

Now if we catch this exception in normal mode, we will only get the md-filename and the message in the console. But if we run it in debug mode, we will get a full python traceback and the context of the tag. And a happy user.

## allow_fail decorator

Often we don't want the whole preprocessor to crash if there are some problems in just one tag of the document. We can easily achieve this using the `allow_fail` decorator, which is included in the `preprocessor_ext` module. Decorate your function, which is then sent to `_process_tags_for_all_files` method:

```python
from foliant.preprocessors.utils.preprocessor_ext import (BasePreprocessorExt,
                                                          allow_fail)


class Preprocessor(BasePreprocessorExt):
  ...
  @allow_fail()
  def _process_tag(match):
    # process the tag
    return processed_string

  def apply(self):
    self._process_tags_for_all_files(func=self._process_tag)
        self.logger.info('Preprocessor applied')
```

Now in case _any_ error occurs in the `_process_tag` function, preprocessor will issue warning, show it to user, save it into log and skip the tag.

The `allow_fail` decorator accepts one argument, the error message, which will be shown to user in case of exception. It defaults to: _Failed to process tag. Skipping._
