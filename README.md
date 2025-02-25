# rasa_composite_entities

A Rasa NLU component for composite entities, developed to be used in the Dialogue Engine of [Dialogue Technologies](https://www.dialogue-technologies.com).

See also [my blog post](https://www.benjaminweigang.com/rasa-nlu-composite-entities/).

**Works with rasa 1.x!**

## Installation

```bash
$ pip install rasa_composite_entities
```

The only external dependency is Rasa NLU itself, which should be installed anyway when you want to use this component.

After installation, the component can be added your pipeline like any other component:

```yaml
language: "en_core_web_md"

pipeline:
- name: "SpacyNLP"
- name: "SpacyTokenizer"
- name: "SpacyFeaturizer"
- name: "CRFEntityExtractor"
- name: "SklearnIntentClassifier"
- name: "rasa_composite_entities.CompositeEntityExtractor"
```

## Usage

Simply add another entry to your training file (in JSON format) defining composite patterns:
```json
"composite_entities": [
  {
    "name": "product_with_attributes",
    "patterns": [
      "@color @product with @pattern",
      "@pattern @color @product"
    ]
  }
],
"common_examples": [
    ...
]
```
Every word starting with a "@" will be considered a placeholder for an entity with that name. The component is agnostic to the origin of entities, you can use anything that Rasa NLU returns as the "entity" field in its messages. This means that you can not only use the entities defined in your common examples, but also numerical entities from duckling etc.

Longer patterns always take precedence over shorter patterns. If a shorter pattern matches entities that would also be matched by a longer pattern, the shorter pattern is ignored.

Patterns are regular expressions! You can use patterns like
```
"composite_entities": [
  {
    "name": "product_with_attributes",
    "patterns": [
      "(?:@pattern\\s+)?(?:@color\\s+)?@product(?:\\s+with @[A-Z,a-z]+)?"
    ]
  }
]
```
to match different variations of entity combinations. Be aware that you may need to properly escape your regexes to produce valid JSON files (in case of this example, you have to escape the backslashes with another backslash).

## Explanation

Composite entities act as containers that group several entities into logical units. Consider the following example phrase:
```
I am looking for a red shirt with stripes and checkered blue shoes.
```
Properly trained, Rasa NLU could return entities like this:
```json
"entities": [
  {
    "start": 19,
    "end": 22,
    "value": "red",
    "entity": "color",
    "confidence": 0.9419322376955782,
    "extractor": "CRFEntityExtractor"
  },
  {
    "start": 23,
    "end": 28,
    "value": "shirt",
    "entity": "product",
    "confidence": 0.9435936216683031,
    "extractor": "CRFEntityExtractor"
  },
  {
    "start": 34,
    "end": 41,
    "value": "stripes",
    "entity": "pattern",
    "confidence": 0.9233923349716401,
    "extractor": "CRFEntityExtractor"
  },
  {
    "start": 46,
    "end": 55,
    "value": "checkered",
    "entity": "pattern",
    "confidence": 0.8877627536275875,
    "extractor": "CRFEntityExtractor"
  },
  {
    "start": 56,
    "end": 60,
    "value": "blue",
    "entity": "color",
    "confidence": 0.6778344517453893,
    "extractor": "CRFEntityExtractor"
  },
  {
    "start": 61,
    "end": 66,
    "value": "shoes",
    "entity": "product",
    "confidence": 0.536797743231954,
    "extractor": "CRFEntityExtractor"
  }
]
```

It's hard to infer exactly what the user is looking for from this output alone. Is he looking for a striped and checkered shirt? Striped and checkered shoes? Or a striped shirt and checkered shoes?

By defining common patterns of entity combinations, we can automatically create entity groups. If we add the composite entity patterns as in the usage example above, the output will be changed to this:
```json
"entities": [
  {
    "confidence": 1.0,
    "entity": "product_with_attributes",
    "extractor": "composite",
    "contained_entities": [
      {
        "start": 19,
        "end": 22,
        "value": "red",
        "entity": "color",
        "confidence": 0.9419322376955782,
        "extractor": "CRFEntityExtractor"
      },
      {
        "start": 23,
        "end": 28,
        "value": "shirt",
        "entity": "product",
        "confidence": 0.9435936216683031,
        "extractor": "CRFEntityExtractor"
      },
      {
        "start": 34,
        "end": 41,
        "value": "stripes",
        "entity": "pattern",
        "confidence": 0.9233923349716401,
        "extractor": "CRFEntityExtractor"
      }
    ]
  },
  {
    "confidence": 1.0,
    "entity": "product_with_attributes",
    "extractor": "composite",
    "contained_entities": [
      {
        "start": 46,
        "end": 55,
        "value": "checkered",
        "entity": "pattern",
        "confidence": 0.8877627536275875,
        "extractor": "CRFEntityExtractor"
      },
      {
        "start": 56,
        "end": 60,
        "value": "blue",
        "entity": "color",
        "confidence": 0.6778344517453893,
        "extractor": "CRFEntityExtractor"
      },
      {
        "start": 61,
        "end": 66,
        "value": "shoes",
        "entity": "product",
        "confidence": 0.536797743231954,
        "extractor": "CRFEntityExtractor"
      }
    ]
  }
]
```

## Example

See the `example` folder for a minimal example that can be trained and tested. To get the output from above, run:
```bash
$ python -m rasa train nlu --out . --nlu train.json --config config_with_composite.yml
$ python -m rasa run --enable-api --model .
$ curl -XPOST localhost:5005/model/parse -d '{"text": "I am looking for a red shirt with stripes and checkered blue shoes"}'
```
If you want to compare this output to the normal Rasa NLU output, use the alternative `config_without_composite.yml` config file.

The component also works when training using the server API:

**HTTP training is currently broken because of API changes in rasa 1.x.
Hopefully, this will soon be fixed!**
```bash
$ python -m rasa run --enable-api --model .
$ curl --request POST --header 'content-type: application/x-yml' --data-binary @train_http.yml --url 'localhost:5000/train?project=test_project'
$ curl -XPOST localhost:5005/model/parse -d '{"text": "I am looking for a red shirt with stripes and checkered blue shoes", "project": "test_project"}'
```

## Caveats

Rasa NLU strips training files of any custom fields, including our "composite_entities" field. For our component to access this information, we have to circumenvent Rasa's train file loading process and get direct access to the raw data.

When training through the Rasa's train script, the train file paths are fetched through the command line arguments.

When training through the HTTP server, we exploit the fact that Rasa NLU creates temporary files containing the raw train data. Be aware that this creates a possible race condition when multiple training processes are executed simultaneously. If a new train process is started before the previous process has reached the CompositeEntityExtractor, there is a chance that the wrong train data will be picked up.

## License

This project is licensed under the MIT License - see the LICENSE.md file for details.
