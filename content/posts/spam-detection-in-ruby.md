---
date: 2024-12-03
title: Spam detection in Ruby
---

## Background

As any good Ruby developer does, I regularly review new projects by [@ankane](https://github.com/ankane).
Most recently I came across [`transformers-ruby`](https://github.com/ankane/transformers-ruby) and have been wondering about how to leverage HuggingFace models.

A use-case finally arose in wanting to flag potential spam messages from users in a Ruby on Rails application. I initially found the [DriftingRuby episode "Detect Spam with AI"](https://www.driftingruby.com/episodes/detect-spam-with-ai) but was dissatisfied with the need to use [a Python microservice](https://github.com/driftingruby/427-detect-spam-with-ai/tree/main/spam-checker).

This is my exploration in how to setup and use a permissively licensed spam detection model in Ruby.

## Setup

Install [`pytorch`](https://pytorch.org/) and configure [`torch-rb`](https://github.com/ankane/torch.rb) to build using the local install:

```sh
brew install pytorch
bundle config build.torch-rb --with-torch-dir=$(brew --prefix)/Cellar/pytorch/2.5.1_1
```

## Example script

```rb
# spamcheck.rb

require 'bundler/inline'

gemfile do
  source 'https://rubygems.org'

  gem "torch-rb"
  gem "transformers-rb"
end

require "torch"
require "transformers"

device =
  case
  when Torch::CUDA.available?
    "cuda"
  when Torch::Backends::MPS.available?
    "mps"
  else
    "cpu"
  end

# DriftingRuby uses `mshenoda/roberta-spam`; however, `transformers-rb` does not yet support RoBERTa models.
# I found the following model which utilizes just BERT and is Apache 2.0 licensed.
model_path = "mrm8488/bert-tiny-finetuned-enron-spam-detection"

embed = Transformers.pipeline("text-classification", model: model_path, device: device)

examples = [
  "Get a free iPhone now!",
  "Hey, can we get together to watch the game tomorrow?",
  "You have won a lottery! To claim the prize, reply with your social security number.",
  "I am attaching the report for your review. Please take a look and let me know if you have any questions.",
  "BlueChew is the better way to get hard. Get your free sample today!",
]

examples.each do |example|
  outputs = embed.(example)
  result = { "LABEL_1" => "Spam", "LABEL_0" => "Ham" }[outputs[:label]]

  puts example
  puts "#{result} (Confidence: #{outputs[:score]})\n\n"
end
```

### Output

```
$ bundle exec ruby spamcheck.rb
Get a free iPhone now!
Spam (Confidence: 0.9982871413230896)

Hey, can we get together to watch the game tomorrow?
Ham (Confidence: 0.9924726486206055)

You have won a lottery! To claim the prize, reply with your social security number.
Spam (Confidence: 0.9984951019287109)

I am attaching the report for your review. Please take a look and let me know if you have any questions.
Ham (Confidence: 0.9982806444168091)

BlueChew is the better way to get hard. Get your free sample today!
Spam (Confidence: 0.9947096109390259)
```

## Follow-ups

- On first run, `transformers-rb` will fetch the model from HuggingFace and then execute it. In a production environment it may be preferrable to move the model fetch into a build step.

- [ankane/informers](https://github.com/ankane/informers) supports [ONNX](https://onnx.ai) models, it may be possible to export the original model used by DriftingRuby to ONNX and then use it. While I don't have any concrete evidence, a [RoBERTa](https://arxiv.org/abs/1907.11692) model seems like it would be better than a [BERT](https://arxiv.org/abs/1810.04805) model just by name.
  - https://huggingface.co/docs/transformers/en/serialization#export-to-onnx
