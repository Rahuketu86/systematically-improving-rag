# Building Production Query Classifiers for RAG Systems

> **Series Overview**: This is the final notebook in our three-part series on systematically analyzing and improving RAG systems. We've discovered patterns and improved clustering—now we'll build production-ready classifiers to act on these insights.

> **Prerequisites**: Complete both "1. Cluster Conversations" and "2. Better Summaries" notebooks first. You'll need `instructor` and `instructor_classify` libraries installed, plus the labeled dataset from our clustering analysis.

## Why This Matters

**Discovery without action is just interesting data.** In our previous notebooks, we systematically analyzed 560 user queries and uncovered clear patterns through improved clustering. We know that three major themes—experiment management, artifact handling, and deployment—account for the majority of user questions. But knowing patterns exist in historical data isn't enough.

**To transform insights into improvements, we need real-time classification.** Specifically, we need to:

1. **Detect Production Drift** - Identify when certain query types suddenly increase, signaling emerging issues
2. **Route Queries Intelligently** - Direct questions to specialized retrieval pipelines based on their category
3. **Prioritize Improvements** - Focus engineering resources on high-volume, low-satisfaction query types
4. **Measure Impact** - Track how changes affect different user segments over time

**Classification bridges the gap between discovery and action**, transforming our topic modeling insights into a production-ready system for continuous RAG improvement.

## What You'll Learn

In this final notebook, you'll discover how to:

1. **Create Production-Ready Classifiers**
   - Build efficient classifiers using the `instructor-classify` framework
   - Generate weak labels automatically for rapid dataset creation
   - Design systematic workflows for human label verification

2. **Achieve High Classification Accuracy**
   - Start with baseline performance and iterate systematically
   - Apply advanced prompting techniques (system prompts, few-shot examples)
   - Measure and visualize improvements using confusion matrices

3. **Scale Classification to Full Datasets**
   - Apply optimized classifiers to thousands of queries efficiently
   - Understand true query distributions across your user base
   - Identify high-impact areas for targeted RAG improvements

## What You'll Achieve

**By the end of this notebook, you'll have built a classifier that achieves 90.9% accuracy**—improving from a 72.7% baseline through systematic prompt engineering. You'll discover that just three categories (artifacts, integrations, visualizations) account for 50% of all user conversations, giving you clear targets for maximum impact improvements.

**More importantly, you'll have a complete methodology for continuous RAG improvement**: discover patterns through clustering → validate with better summaries → monitor continuously through classification → prioritize improvements based on real usage data.

## From Reactive to Proactive RAG Systems

**Most RAG systems improve reactively**—waiting for user complaints or noticing obvious failures. **This series shows you how to build proactively improving systems** that identify problems before users complain and prioritize fixes based on systematic analysis rather than the loudest feedback.

By the end of this notebook, you'll have moved from "we think users struggle with X" to "we know 20% of users need help with artifacts, 15% with integrations, and 14% with visualizations—and we can automatically detect and route these queries for specialized handling."

## What You'll Learn

In this notebook, you'll discover how to:

1. **Generate Weak Labels and Create a Golden Dataset**
   - Create an initial classifier using the instructor-classify framework
   - Generate preliminary labels for your conversation dataset
   - Use app.py to review and correct weak labels for a high-quality labeled dataset

2. **Iteratively Improve Classification Accuracy**
   - Start with a simple baseline classifier
   - Enhance performance with few-shot examples and system prompts
   - Measure improvements using confusion matrices and accuracy metrics

3. **Analyze Query Distribution in Your Dataset**
   - Apply your optimized classifier to the full dataset
   - Understand the prevalence of different query types
   - Identify high-impact areas for RAG system improvements

Rather than trying to replicate all the nuanced clusters from our topic modeling, we'll focus on three high-impact categories that emerged from our analysis:

1. Artifacts - Questions about creating, versioning, and managing W&B artifacts
2. Integrations - Questions about integrating W&B with specific libraries and frameworks
3. Visualisations - Questions about creating charts, dashboards, and visual analysis
4. Other - General queries that don't fit the specialized categories above

By the end of this notebook, you'll have moved from "we discovered these patterns exist" to "we can automatically detect and act on these patterns in production."

## Defining Our Classifier

Our topic modeling revealed several distinct clusters of user queries, with three major topics accounting for the majority of questions:

1. Users seeking help with experiment tracking and metrics logging
2. Users trying to manage artifacts and data versioning
3. Users needing assistance with integrations and deployment

In this notebook, we'll show how we might build a classifier which can identify queries related to creating, managing and versioning weights and biases artifacts, questions about integrations as well as visualisations of data that's been logged.

1. First we'll define a simple classifier using `instructor-classify` that will take in a query and document pair and output a suggested category for it
2. Then we'll see a few examples of the `instructor-classify` library in action
3. Lastly, we'll then generate a set of initial weak labels using this simple classifier before exporting it to a file for manual annotation using our `app.py` file.

Let's get started with the `instructor-classify` library

```python
from instructor_classify.schema import LabelDefinition, ClassificationDefinition

artifact_label = LabelDefinition(
    label="artifact",
    description="This is a user query and document pair which is about creating, versioning and managing weights and biases artifacts.",
)

integrations_label = LabelDefinition(
    label="integrations",
    description="this is a user query and document pair which is concerned with how we can integrate weights and biases with specific libraries"
)

visualisation_label = LabelDefinition(
    label="visualisation",
    description="This is a user query and document pair which is concerned about how we can visualise the data that we've logged with weights and biases"
)

other_label = LabelDefinition(
    label="other",
    description="Use this label for other query types which don't belong to any of the other defined categories that you have been provided with",
)


classification_def = ClassificationDefinition(
    system_message="You're an expert at classifying a user and document pair. Look closely at the user query and determine what the query is about and how the document helps answer it. Then classify it according to the label(s) above. Classify irrelevant ones as Other",
    label_definitions=[artifact_label, other_label, visualisation_label, integrations_label],
)

```

This structure makes it easy to define multiple categories in a way that's clear to both humans and LLMs. It provides explicit definitions of what each category means, making it easier for the model to make accurate predictions.

We also support exporing this configuration to a `.yaml` format for easy use if you're working with domain experts for easy collaboration.

### A Simple Example

Let's now see how `instructor-classify` works under the hood. We'll do so by passing in 4 sample queries and seeing how our classifier is able to deal with these test cases

```python
import instructor
from instructor_classify.classify import Classifier
from openai import OpenAI

client = instructor.from_openai(OpenAI())
classifier = (
    Classifier(classification_def).with_client(client).with_model("gpt-4.1-mini")
)

# Make a prediction
result = classifier.predict("How do I version a weights and biases artifact?")
print(f"Classification: {result}")  # Should output "artifact";

result_2 = classifier.predict("What is the square root of 9?")
print(f"Classification: {result_2}")  # Should output "not_artifact"
```

    Classification: label='artifact'
    Classification: label='other'

`instructor-classify` exposes a `batch_predict` function which parallelises this operation for us so that we can run evaluations efficiently over large datasets. Let's see it in action below with some test cases

```python
tests = [
    "How do I version a weights and biases artifact?",
    "What is the square root of 9?",
    "How do I integrate weights and biases with pytorch?",
    "What are some best practices when using wandb?",
    "How can I visualise my training runs?",
]

labels = [
    "artifact",
    "other",
    "integrations",
    "other",
    "visualisation"
]

results = classifier.batch_predict(tests)
for query, result, label in zip(tests, results, labels):
    print(f"Query: {query}\nClassification: {result}\nExpected: {label}\n")

```

    classify:   0%|          | 0/5 [00:00<?, ?it/s]



    Query: How do I version a weights and biases artifact?
    Classification: label='artifact'
    Expected: artifact

    Query: What is the square root of 9?
    Classification: label='other'
    Expected: other

    Query: How do I integrate weights and biases with pytorch?
    Classification: label='integrations'
    Expected: integrations

    Query: What are some best practices when using wandb?
    Classification: label='other'
    Expected: other

    Query: How can I visualise my training runs?
    Classification: label='visualisation'
    Expected: visualisation

### Creating Weak Labels

What is a weak label? A weak label is an automatically generated label that might be incorrect, but is "good enough" to start with. Think of it like a rough first draft - it gives you a starting point that you can then review and correct.

In real-world RAG systems, manually labeling thousands of queries is prohibitively time-consuming. Instead of having humans label every single query from scratch, weak labeling lets us use an AI classifier to do the initial work.

Here's how it works:

1. Our classifier automatically assigns labels to hundreds of queries in minutes
2. We review these "weak labels" and ask annotators to simply accept/reject the labels
3. This allows us to create gold label datasets much faster and efficiently that we would be able to do so manually.

This approach has several key advantages:

1. Speed: We can process hundreds or thousands of examples in minutes rather than hours
2. Consistency: The classifier applies the same criteria across all examples
3. Scalability: The process can be applied to continuously growing datasets

The process follows a virtuous cycle: our classifier generates weak labels → humans verify a subset → verified labels improve the classifier → better classifier generates more accurate weak labels.

To help you try this, we've created a simple UI using streamlit where we can then either approve or reject these labels. You can run this at `streamlit run app.py`.

```python
import json

with open("./data/conversations.json") as f:
    conversations_raw = json.load(f)

texts = [
    {
        "query": item["query"],
        "matching_document": item["matching_document"],
        "query_id": item["query_id"],
    }
    for item in conversations_raw
]

results = classifier.batch_predict(texts[:110])
```

```python
with open("./data/generated.jsonl","w+") as f:
    for item,result in zip(conversations_raw,results):
        f.write(json.dumps({
            "query": item["query"],
            "matching_document": item["matching_document"],
            "query_id": item["query_id"],
            "labels": result.label
        })+"\n")
```

## Evaluating Our Classifier

We've labelled a dataset of ~100 items our of our 560 conversations ahead of time. If you'd like to label more datasets, we've provided an `app.py` file which you can run using the command `streamlit run app.py`.

We'll be splitting this into a test and validation split. We'll be using the `validation` split to iterate on our prompt and experiment with different few shot examples before using the `test` split to validate our classifier's performance.

We'll be using a 70-30 split with 70% of our data used for validation and 30% used for testing our final classifier.

```python
import json
import random

with open("./data/labels.jsonl") as f:
    conversations_labels = [json.loads(line) for line in f]

# Set random seed for reproducibility
random.seed(42)

# Shuffle the data
random.shuffle(conversations_labels)

# Calculate split index
split_idx = int(len(conversations_labels) * 0.7)

# Split into validation and test sets
val_set = conversations_labels[:split_idx]
test_set = conversations_labels[split_idx:]

print(f"Validation set size: {len(val_set)}")
print(f"Test set size: {len(test_set)}")
```

    Validation set size: 77
    Test set size: 33

### Determining a baseline

Let's now calculate a baseline and see how well our initial classification model performs.

```python
val_text = [
    f"bQuery: {item['query']}\n Retrieved Document: {item['matching_document']}"
    for item in val_set
]
val_labels = [item["labels"] for item in val_set]

test_text = [
    f"Query: {item['query']}\n Retrieved Document: {item['matching_document']}"
    for item in test_set
]
test_labels = [item["labels"] for item in test_set]
```

Let's now define a function which runs the classifier on the validation set and the test set to see our initial starting point. We'll look at some of the failure cases and then iterately improve our classifier.

```python
from sklearn.metrics import confusion_matrix
from instructor_classify.classify import Classifier

def predict_and_evaluate(classifier: Classifier, texts: list[str], labels:list[str]):
    predictions = classifier.batch_predict(texts)
    pred_labels = [p.label for p in predictions]

    return {
        "accuracy": sum(pred == label for pred, label in zip(pred_labels, labels)) / len(predictions),
        "queries": texts,
        "labels": labels,
        "predictions": pred_labels
    }

classifier = (
    Classifier(classification_def).with_client(client).with_model("gpt-4.1-mini")
)
predictions = predict_and_evaluate(classifier, val_text, val_labels)
predictions["accuracy"]
```

    0.7272727272727273

```python
import matplotlib.pyplot as plt
from sklearn.metrics import ConfusionMatrixDisplay

# Get unique labels
unique_labels = ["artifact", "other", "visualisation", "integrations"]

# Convert predictions and true labels to label indices
y_true = [unique_labels.index(label) for label in predictions["labels"]]
y_pred = [unique_labels.index(label) for label in predictions["predictions"]]

# Calculate single confusion matrix for all categories
conf_matrix = confusion_matrix(y_true, y_pred)

# Plot confusion matrix
plt.figure(figsize=(10, 8))
disp = ConfusionMatrixDisplay(confusion_matrix=conf_matrix, display_labels=unique_labels)
disp.plot()
plt.title('Confusion Matrix for All Categories')
plt.tight_layout()
plt.show()
```

    <Figure size 1000x800 with 0 Axes>

![png](extra_kura_03_classifiers_files/extra_kura_03_classifiers_18_1.png)

Let's now see how it looks like when we run it on our test set

```python
test_predictions = predict_and_evaluate(classifier,test_text, test_labels)
test_predictions["accuracy"]
```

    0.696969696969697

```python
import matplotlib.pyplot as plt
from sklearn.metrics import ConfusionMatrixDisplay

# Get unique labels
unique_labels = ["artifact", "other", "visualisation", "integrations"]

# Convert predictions and true labels to label indices
y_true = [unique_labels.index(label) for label in test_predictions["labels"]]
y_pred = [unique_labels.index(label) for label in test_predictions["predictions"]]

# Calculate single confusion matrix for all categories
conf_matrix = confusion_matrix(y_true, y_pred)

# Plot confusion matrix
plt.figure(figsize=(10, 8))
disp = ConfusionMatrixDisplay(confusion_matrix=conf_matrix, display_labels=unique_labels)
disp.plot()
plt.title('Confusion Matrix for All Categories')
plt.tight_layout()
plt.show()
```

    <Figure size 1000x800 with 0 Axes>

![png](extra_kura_03_classifiers_files/extra_kura_03_classifiers_21_1.png)

### Looking at Edge Cases

Let's now print out some of the errors that our model made in classifying our user queries

```python
for prediction,label,query in zip(test_predictions["predictions"],test_predictions["labels"],test_predictions["queries"]):
    if label != prediction:
        print(f"Label: {label}")
        print(f"Prediction: {prediction}")
        print(f"Query: {query}")
        print("=====")

```

    Label: other
    Prediction: artifact
    Query: Query: machine learning model tracking
     Retrieved Document: ## Track a model

    Track a model, the model's dependencies, and other information relevant to that model with the W&B Python SDK.

    Under the hood, W&B creates a lineage of model artifact that you can view with the W&B App UI or programmatically with the W&B Python SDK. See the Create model lineage map for more information.

    ## How to log a model

    Use the `run.log_model` API to log a model. Provide the path where your model files are saved to the `path` parameter. The path can be a local file, directory, or reference URI to an external bucket such as `s3://bucket/path`.

    Optionally provide a name for the model artifact for the `name` parameter. If `name` is not specified, W&B uses the basename of the input path prepended with the run ID.

    Copy and paste the proceeding code snippet. Ensure to replace values enclosed in `<>` with your own.
    =====
    Label: other
    Prediction: artifact
    Query: Query: can you provide a bit more clarity on the difference between setting `resume` in `wandb.init` to `allow` vs. `auto`?

    I guess the difference has to do with whether the previous run crashed or not. I guess if the run didn't crash, `auto` may overwrite if there's matching `id`?
     Retrieved Document: | `resume` | (bool, str, optional) Sets the resuming behavior. Options: `"allow"`, `"must"`, `"never"`, `"auto"` or `None`. Defaults to `None`. Cases: - `None` (default): If the new run has the same ID as a previous run, this run overwrites that data. - `"auto"` (or `True`): if the previous run on this machine crashed, automatically resume it. Otherwise, start a new run. - `"allow"`: if id is set with `init(id="UNIQUE_ID")` or `WANDB_RUN_ID="UNIQUE_ID"` and it is identical to a previous run, wandb will automatically resume the run with that id. Otherwise, wandb will start a new run. - `"never"`: if id is set with `init(id="UNIQUE_ID")` or `WANDB_RUN_ID="UNIQUE_ID"` and it is identical to a previous run, wandb will crash. - `"must"`: if id is set with `init(id="UNIQUE_ID")` or `WANDB_RUN_ID="UNIQUE_ID"` and it is identical to a previous run, wandb will automatically resume the run with the id. Otherwise, wandb will crash. See our guide to resuming runs for more. |
    | `reinit` | (bool, optional) Allow multiple `wandb.init()` calls in the same process. (default: `False`) |
    | `magic` | (bool, dict, or str, optional) The bool controls whether we try to auto-instrument your script, capturing basic details of your run without you having to add more wandb code. (default: `False`) You can also pass a dict, json string, or yaml filename. |
    | `config_exclude_keys` | (list, optional) string keys to exclude from `wandb.config`. |
    | `config_include_keys` | (list, optional) string keys to include in `wandb.config`. |
    =====
    Label: other
    Prediction: visualisation
    Query: Query: W&B report sharing
     Retrieved Document: ---

    ## description: Collaborate and share W&B Reports with peers, co-workers, and your team.

    # Collaborate on reports

    Collaborate and Share W&B Reports

    Once you have saved a report, you can select the **Share** button to collaborate. A draft copy of the report is created when you select the **Edit** button. Draft reports auto-save. Select **Save to report** to publish your changes to the shared report.

    A warning notification will appear if an edit conflict occurs. This can occur if you and another collaborator edit the same report at the same time. The warning notification will guide you to resolve potential edit conflicts.

    ### Comment on reports

    Click the comment button on a panel in a report to add a comment directly to that panel.

    ### Who can edit and share reports?

    Reports that are created within an individual's private project is only visible to that user. The user can share their project to a team or to the public.
    =====
    Label: other
    Prediction: artifact
    Query: Query: Tracking and comparing LLM experiments in Weights & Biases
     Retrieved Document: ### Using Weights & Biases to track experiments

    Experimenting with prompts, function calling and response model schema is critical to get good results. As LLM Engineers, we will be methodical and use Weights & Biases to track our experiments.

    Here are a few things you should consider logging:

    1. Save input and output pairs for later analysis
    2. Save the JSON schema for the response\_model
    3. Having snapshots of the model and data allow us to compare results over time, and as we make changes to the model we can see how the results change.

    This is particularly useful when we might want to blend a mix of synthetic and real data to evaluate our model. We will use the `wandb` library to track our experiments and save the results to a dashboard.
    =====
    Label: other
    Prediction: artifact
    Query: Query: how to define W&B sweep in YAML
     Retrieved Document: ## Add W&B to your code
    #### Training script with W&B Python SDK
    To create a W&B Sweep, we first create a YAML configuration file. The configuration file contains he hyperparameters we want the sweep to explore. In the proceeding example, the batch size (`batch_size`), epochs (`epochs`), and the learning rate (`lr`) hyperparameters are varied during each sweep.

    ```
    # config.yaml
    program: train.py
    method: random
    name: sweep
    metric:
    goal: maximize
    name: val\_acc
    parameters:
    batch\_size:
    values: [16,32,64]
    lr:
    min: 0.0001
    max: 0.1
    epochs:
    values: [5, 10, 15]

    ```

    For more information on how to create a W&B Sweep configuration, see Define sweep configuration.

    Note that you must provide the name of your Python script for the `program` key in your YAML file.

    Next, we add the following to the code example:
    =====
    Label: other
    Prediction: integrations
    Query: Query: logging distributed training wandb
     Retrieved Document: ## Experiments FAQ
    #### How can I use wandb with multiprocessing, e.g. distributed training?

    If your training program uses multiple processes you will need to structure your program to avoid making wandb method calls from processes where you did not run `wandb.init()`.\
    \
    There are several approaches to managing multiprocess training:

    1. Call `wandb.init` in all your processes, using the group keyword argument to define a shared group. Each process will have its own wandb run and the UI will group the training processes together.
    2. Call `wandb.init` from just one process and pass data to be logged over multiprocessing queues.

    :::info
    Check out the Distributed Training Guide for more detail on these two approaches, including code examples with Torch DDP.
    :::
    =====
    Label: other
    Prediction: artifact
    Query: Query: What does setting the 'resume' parameter to 'allow' do in wandb.init?
     Retrieved Document: ## Resume Runs
    #### Resume Guidance
    ##### Automatic and controlled resuming

    Automatic resuming only works if the process is restarted on top of the same filesystem as the failed process. If you can't share a filesystem, we allow you to set the `WANDB_RUN_ID`: a globally unique string (per project) corresponding to a single run of your script. It must be no longer than 64 characters. All non-word characters will be converted to dashes.

    ```
    # store this id to use it later when resuming
    id = wandb.util.generate\_id()
    wandb.init(id=id, resume="allow")
    # or via environment variables
    os.environ["WANDB\_RESUME"] = "allow"
    os.environ["WANDB\_RUN\_ID"] = wandb.util.generate\_id()
    wandb.init()

    ```

    If you set `WANDB_RESUME` equal to `"allow"`, you can always set `WANDB_RUN_ID` to a unique string and restarts of the process will be handled automatically. If you set `WANDB_RESUME` equal to `"must"`, W&B will throw an error if the run to be resumed does not exist yet instead of auto-creating a new run.
    =====
    Label: other
    Prediction: artifact
    Query: Query: wandb.init() code saving
     Retrieved Document: ---

    ## displayed\_sidebar: default

    # Code Saving

    By default, we only save the latest git commit hash. You can turn on more code features to compare the code between your experiments dynamically in the UI.

    Starting with `wandb` version 0.8.28, we can save the code from your main training file where you call `wandb.init()`. This will get sync'd to the dashboard and show up in a tab on the run page, as well as the Code Comparer panel. Go to your settings page to enable code saving by default.

    ## Save Library Code

    When code saving is enabled, wandb will save the code from the file that called `wandb.init()`. To save additional library code, you have two options:

    * Call `wandb.run.log_code(".")` after calling `wandb.init()`
    * Pass a settings object to `wandb.init` with code\_dir set: `wandb.init(settings=wandb.Settings(code_dir="."))`

    ## Code Comparer

    ## Jupyter Session History

    ## Jupyter diffing
    =====
    Label: other
    Prediction: artifact
    Query: Query: How does wandb.save function and what are its use cases?
     Retrieved Document: ## Save your machine learning model
    * Use wandb.save(filename).
    * Put a file in the wandb run directory, and it will get uploaded at the end of the run.

    If you want to sync files as they're being written, you can specify a filename or glob in wandb.save.

    Here's how you can do this in just a few lines of code. See [this colab](https://colab.research.google.com/drive/1pVlV6Ua4C695jVbLoG-wtc50wZ9OOjnC) for a complete example.

    ```
    # "model.h5" is saved in wandb.run.dir & will be uploaded at the end of training
    model.save(os.path.join(wandb.run.dir, "model.h5"))

    # Save a model file manually from the current directory:
    wandb.save('model.h5')

    # Save all files that currently exist containing the substring "ckpt":
    wandb.save('../logs/*ckpt*')

    # Save any files starting with "checkpoint" as they're written to:
    wandb.save(os.path.join(wandb.run.dir, "checkpoint*"))
    ```
    =====
    Label: other
    Prediction: artifact
    Query: Query: best practices for tracking experiments in Weights & Biases
     Retrieved Document: ## Create an Experiment
    ### Best Practices

    The following are some suggested guidelines to consider when you create experiments:

    1. **Config**: Track hyperparameters, architecture, dataset, and anything else you'd like to use to reproduce your model. These will show up in columns— use config columns to group, sort, and filter runs dynamically in the app.
    2. **Project**: A project is a set of experiments you can compare together. Each project gets a dedicated dashboard page, and you can easily turn on and off different groups of runs to compare different model versions.
    3. **Notes**: A quick commit message to yourself. The note can be set from your script. You can edit notes at a later time on the Overview section of your project's dashboard on the W&B App.
    4. **Tags**: Identify baseline runs and favorite runs. You can filter runs using tags. You can edit tags at a later time on the Overview section of your project's dashboard on the W&B App.
    =====

When exmaining our confusion matrices in detail, we observe a consistent pattern of misclassification in the "other" category where our classifier frequently misidentifies these queries by assigning them to one of our specific categories.

Looking at the classification errors, we can identify several patterns

1. `Context Confusion` : The model tends to ignore the user's specific question but instead gets confused by the retrieved document. If a document contains specific bits of information about an artifact, even if the user's question is simply a general question.
2. `Over-Eagerness` : The model tends to prefer assigning specialised categories rathern than the more general "other" category, even when evidence is limited. This results in false positives for our specialised categories.

To address these issues, we'll need to carefully craft our prompts to help the model better distinguish between general W&B functionality and specific feature categories.

By combining improved system prompts with strategically selected few-shot examples, we can guide the model to pay closer attention to the user's actual intent rather than being misled by terminology in the retrieved documents.

Our next steps will focus on implementing these improvements and measuring their impact on classification accuracy, particularly for the challenging "other" category where most of our errors occur.

## Improving Our Classifier

Our baseline classifier achieved approximately 73% accuracy on the validation set, but the confusion matrices revealed significant challenges with the "other" category.

To address these issues, we'll take a systematic approach to enhancement:

1. Refining system prompts to provide clearer boundaries between categories and explicitly instruct the model on how to handle ambiguous cases
2. Adding few-shot examples that demonstrate the correct handling of edge cases, particularly for general queries that mention specialized terms

Let's get started and see how to do so.

### System Prompt

The first improvement we'll implement is a more precise system prompt. Our error analysis showed that the model frequently misclassifies general queries as specialized categories when the retrieved document mentions features like artifacts or visualisations.

By providing explicit instructions about how to prioritize the user's query over the retrieved document and establishing clearer category boundaries, we can help the model make more accurate distinctions. We'll also provide a clear description of what each category represents so taht the model can make more accurate distinctions.

```python
import instructor
from instructor_classify.classify import Classifier
from instructor_classify.schema import LabelDefinition, ClassificationDefinition
from openai import OpenAI

client = instructor.from_openai(OpenAI())

artifact_label = LabelDefinition(
    label="artifact",
    description="This is a user query and document pair which is about creating, versioning and managing weights and biases artifacts.",
)

integrations_label = LabelDefinition(
    label="integrations",
    description="this is a user query and document pair which is concerned with how we can integrate weights and biases with specific libraries"
)

visualisation_label = LabelDefinition(
    label="visualisation",
    description="This is a user query and document pair which is concerned about how we can visualise the data that we've logged with weights and biases"
)

other_label = LabelDefinition(
    label="other",
    description="Use this label for other query types which don't belong to any of the other defined categories that you have been provided with",
)


classification_def_w_system_prompt = ClassificationDefinition(
    system_message="""
You are a world class classifier. You are given a user query and a document.

Look closely at the user query and determine what the user's query is about. You will also be provided with a document which is relevant to the user's query. It might contain additional information that's not relevant to the user's query so when deciding on the category, only consider the parts that are relevant to answering the user's specific question.

Here are the categories you can choose from:

1. Artifacts - questions about creating, versioning and managing weights and biases artifacts. Note that in W&B there are two ways to store data - Artifacts and Files. Only consider queries about artifacts when they explicitly mention the usage of Artifacts with W&B.
2. Integrations - questions about integrating weights and biases with specific libraries.
3. Visualisation - questions about visualising the data that we've logged with weights and biases
4. Other - queries that don't fit the above categories

Choose a single category for the user query and document pair that's the best match""",
    label_definitions=[artifact_label, other_label, visualisation_label, integrations_label],
)

```

```python
client = instructor.from_openai(OpenAI())
classifier_v2 = (
    Classifier(classification_def_w_system_prompt).with_client(client).with_model("gpt-4.1-mini")
)
```

```python
predictions_system_prompt = predict_and_evaluate(classifier_v2, val_text, val_labels)
predictions_system_prompt["accuracy"]
```

    0.8051948051948052

Let's now see how our model performs by using a confusion matrix

```python
import matplotlib.pyplot as plt
from sklearn.metrics import ConfusionMatrixDisplay

# Get unique labels
unique_labels = ["artifact", "other", "visualisation", "integrations"]

# Convert predictions and true labels to label indices
y_true = [unique_labels.index(label) for label in predictions_system_prompt["labels"]]
y_pred = [unique_labels.index(label) for label in predictions_system_prompt["predictions"]]

# Calculate single confusion matrix for all categories
conf_matrix = confusion_matrix(y_true, y_pred)

# Plot confusion matrix
plt.figure(figsize=(10, 8))
disp = ConfusionMatrixDisplay(confusion_matrix=conf_matrix, display_labels=unique_labels)
disp.plot()
plt.title('Confusion Matrix for All Categories')
plt.tight_layout()
plt.show()
```

    <Figure size 1000x800 with 0 Axes>

![png](extra_kura_03_classifiers_files/extra_kura_03_classifiers_30_1.png)

It seems that with this new prompt, we've seen a roughly 16% improvement in our accuracy. One major issue seems to be classifying queries as the `other` category well. Let's visualise some of these queries

```python
for prediction,label,query in zip(predictions_system_prompt["predictions"],predictions_system_prompt["labels"],predictions_system_prompt["queries"]):
    if label != prediction:
        print(f"Label: {label}")
        print(f"Prediction: {prediction}")
        print(f"Query: {query}")
        print("=====")
```

    Label: other
    Prediction: visualisation
    Query: Query: Examples of logging images in Wandb
     Retrieved Document: ## Log Media & Objects
    ### Images
    ```
    images = wandb.Image(image\_array, caption="Top: Output, Bottom: Input")

    wandb.log({"examples": images})

    ```

    We assume the image is gray scale if the last dimension is 1, RGB if it's 3, and RGBA if it's 4. If the array contains floats, we convert them to integers between `0` and `255`. If you want to normalize your images differently, you can specify the `mode` manually or just supply a `PIL.Image`, as described in the "Logging PIL Images" tab of this panel.

    For full control over the conversion of arrays to images, construct the `PIL.Image` yourself and provide it directly.

    ```
    images = [PIL.Image.fromarray(image) for image in image\_array]

    wandb.log({"examples": [wandb.Image(image) for image in images]})

    ```

    For even more control, create images however you like, save them to disk, and provide a filepath.

    ```
    im = PIL.fromarray(...)
    rgb\_im = im.convert("RGB")
    rgb\_im.save("myimage.jpg")

    wandb.log({"example": wandb.Image("myimage.jpg")})

    ```
    =====
    Label: other
    Prediction: integrations
    Query: Query: How to structure Weights & Biases runs for hyperparameter tuning?
     Retrieved Document: ## Whats Next? Hyperparameters with Sweeps

    We tried out two different hyperparameter settings by hand. You can use Weights & Biases Sweeps to automate hyperparameter testing and explore the space of possible models and optimization strategies.

    ## Check out Hyperparameter Optimization in TensorFlow uisng W&B Sweep $\rightarrow$

    Running a hyperparameter sweep with Weights & Biases is very easy. There are just 3 simple steps:

    1. **Define the sweep:** We do this by creating a dictionary or a YAML file that specifies the parameters to search through, the search strategy, the optimization metric et all.
    2. **Initialize the sweep:**
    `sweep_id = wandb.sweep(sweep_config)`
    3. **Run the sweep agent:**
    `wandb.agent(sweep_id, function=train)`

    And voila! That's all there is to running a hyperparameter sweep! In the notebook below, we'll walk through these 3 steps in more detail.
    =====
    Label: other
    Prediction: integrations
    Query: Query: What are common issues when logging distributed training with wandb?
     Retrieved Document: ## Train model with DDP
    The preceding image demonstrates the W&B App UI dashboard. On the sidebar we see two experiments. One labeled 'null' and a second (bound by a yellow box) called 'DPP'. If you expand the group (select the Group dropdown) you will see the W&B Runs that are associated to that experiment.

    ### Use W&B Service to avoid common distributed training issues.

    There are two common issues you might encounter when using W&B and distributed training:

    1. **Hanging at the beginning of training** - A `wandb` process can hang if the `wandb` multiprocessing interferes with the multiprocessing from distributed training.
    2. **Hanging at the end of training** - A training job might hang if the `wandb` process does not know when it needs to exit. Call the `wandb.finish()` API at the end of your Python script to tell W&B that the Run finished. The wandb.finish() API will finish uploading data and will cause W&B to exit.

    ### Enable W&B Service

    ### Example use cases for multiprocessing
    =====
    Label: other
    Prediction: integrations
    Query: Query: What are the common issues when logging distributed training with wandb?
     Retrieved Document: ## Train model with DDP
    The preceding image demonstrates the W&B App UI dashboard. On the sidebar we see two experiments. One labeled 'null' and a second (bound by a yellow box) called 'DPP'. If you expand the group (select the Group dropdown) you will see the W&B Runs that are associated to that experiment.

    ### Use W&B Service to avoid common distributed training issues.

    There are two common issues you might encounter when using W&B and distributed training:

    1. **Hanging at the beginning of training** - A `wandb` process can hang if the `wandb` multiprocessing interferes with the multiprocessing from distributed training.
    2. **Hanging at the end of training** - A training job might hang if the `wandb` process does not know when it needs to exit. Call the `wandb.finish()` API at the end of your Python script to tell W&B that the Run finished. The wandb.finish() API will finish uploading data and will cause W&B to exit.

    ### Enable W&B Service

    ### Example use cases for multiprocessing
    =====
    Label: other
    Prediction: integrations
    Query: Query: Are there any best practices for using wandb in a distributed training environment?
     Retrieved Document: ## Add wandb to Any Library
    #### Distributed Training

    For frameworks supporting distributed environments, you can adapt any of the following workflows:

    * Detect which is the “main” process and only use `wandb` there. Any required data coming from other processes must be routed to the main process first. (This workflow is encouraged).
    * Call `wandb` in every process and auto-group them by giving them all the same unique `group` name

    See Log Distributed Training Experiments for more details
    =====
    Label: other
    Prediction: integrations
    Query: Query: How to use IAM roles with SageMaker for training job access control?
     Retrieved Document: ## Set up for SageMaker
    ### Prerequisites
    1. **Setup SageMaker in your AWS account.** See the SageMaker Developer guide for more information.
    2. **Create an Amazon ECR repository** to store images you want to execute on Amazon SageMaker. See the Amazon ECR documentation for more information.
    3. **Create an Amazon S3 buckets** to store SageMaker inputs and outputs for your SageMaker training jobs. See the Amazon S3 documentation for more information. Make note of the S3 bucket URI and directory.
    4. **Create IAM execution role.** The role used in the SageMaker training job requires the following permissions to work. These permissions allow for logging events, pulling from ECR, and interacting with input and output buckets. (Note: if you already have this role for SageMaker training jobs, you do not need to create it again.)
    IAM role policy
    ```
    {
    "Version": "2012-10-17",
    "Statement": [
    {
    "Effect": "Allow",
    "Action": [
    "cloudwatch:PutMetricData",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "logs:CreateLogGroup",
    "logs:DescribeLogStreams",
    "ecr:GetAuthorizationToken"
    ],
    "Resource": "\*"
    },
    {
    "Effect": "Allow",
    "Action": [
    "s3:ListBucket"
    ],
    "Resource": [
    "arn:aws:s3:::<input-bucket>"
    ]
    },
    {
    "Effect": "Allow",
    "Action": [
    "s3:GetObject",
    "s3:PutObject"
    ],
    "Resource": [
    "arn:aws:s3:::<input-bucket>/<object>",
    "arn:aws:s3:::<output-bucket>/<path>"
    ]
    },
    {
    "Effect": "Allow",
    "Action": [
    "ecr:BatchCheckLayerAvailability",
    "ecr:GetDownloadUrlForLayer",
    "ecr:BatchGetImage"
    ],
    "Resource": "arn:aws:ecr:<region>:<account-id>:repository/<repo>"
    }
    ]
    }

    ```
    =====
    Label: other
    Prediction: integrations
    Query: Query: wandb setup
     Retrieved Document: ## 🚀 Setup

    Start out by installing the experiment tracking library and setting up your free W&B account:

    1. Install with `!pip install`
    2. `import` the library into Python
    3. `.login()` so you can log metrics to your projects

    If you've never used Weights & Biases before,
    the call to `login` will give you a link to sign up for an account.
    W&B is free to use for personal and academic projects!

    ```
    !pip install wandb -Uq

    ```

    ```
    import wandb

    ```

    ```
    wandb.login()

    ```
    =====
    Label: other
    Prediction: visualisation
    Query: Query: Weights & Biases features for LLM developers
     Retrieved Document: **Weights & Biases Prompts** is a suite of LLMOps tools built for the development of LLM-powered applications.

    Use W&B Prompts to visualize and inspect the execution flow of your LLMs, analyze the inputs and outputs of your LLMs, view the intermediate results and securely store and manage your prompts and LLM chain configurations.

    #### 🪄 View Prompts In Action

    **In this notebook we will demostrate W&B Prompts:**

    * Using our 1-line LangChain integration
    * Using our Trace class when building your own LLM Pipelines

    See here for the full W&B Prompts documentation

    ## Installation

    ```
    !pip install "wandb>=0.15.4" -qqq
    !pip install "langchain>=0.0.218" openai -qqq

    ```

    ```
    import langchain
    assert langchain.__version__ >= "0.0.218", "Please ensure you are using LangChain v0.0.188 or higher"

    ```

    ## Setup

    This demo requires that you have an OpenAI key

    # W&B Prompts

    W&B Prompts consists of three main components:

    **Trace table**: Overview of the inputs and outputs of a chain.
    =====
    Label: other
    Prediction: artifact
    Query: Query: log prompts wandb
     Retrieved Document: def log_index(vector_store_dir: str, run: "wandb.run"):
        """Log a vector store to wandb

        Args:
            vector_store_dir (str): The directory containing the vector store to log
            run (wandb.run): The wandb run to log the artifact to.
        """
        index_artifact = wandb.Artifact(name="vector_store", type="search_index")
        index_artifact.add_dir(vector_store_dir)
        run.log_artifact(index_artifact)

    def log_prompt(prompt: dict, run: "wandb.run"):
        """Log a prompt to wandb

        Args:
            prompt (str): The prompt to log
            run (wandb.run): The wandb run to log the artifact to.
        """
        prompt_artifact = wandb.Artifact(name="chat_prompt", type="prompt")
        with prompt_artifact.new_file("prompt.json") as f:
            f.write(json.dumps(prompt))
        run.log_artifact(prompt_artifact)
    =====
    Label: other
    Prediction: visualisation
    Query: Query: Weights & Biases GPU utilization
     Retrieved Document: ## Monitor & Improve GPU Usage for Model Training
    #### 1. Measure your GPU usage consistently over your entire training runs
    You can’t improve GPU usage without measuring it.  It’s not hard to take a snapshot of your usage with useful tools like nvidia-smi, but a simple way to find issues is to track usage over time.  Anyone can turn on system monitoring in the background, which will track GPU, CPU, memory usage etc over time by adding two lines to their code:

    ```
    import wandb
    wandb.init()
    ```

    The wandb.init() function will create a lightweight child process that will collect system metrics and send them to a wandb server where you can look at them and compare across runs with graphs like these:

    The danger of taking a single measurement is that GPU usage can change over time.  This is a common pattern we see where our user Boris is training an RNN; mid-training, his usage plummets from 80 percent to around 25 percent.
    =====
    Label: other
    Prediction: integrations
    Query: Query: building LLM-powered apps with W&B
     Retrieved Document: ## Prompts for LLMs

    W&B Prompts is a suite of LLMOps tools built for the development of LLM-powered applications. Use W&B Prompts to visualize and inspect the execution flow of your LLMs, analyze the inputs and outputs of your LLMs, view the intermediate results and securely store and manage your prompts and LLM chain configurations.

    ## Use Cases

    W&B Prompts provides several solutions for building and monitoring LLM-based apps. Software developers, prompt engineers, ML practitioners, data scientists, and other stakeholders working with LLMs need cutting-edge tools to:

    * Explore and debug LLM chains and prompts with greater granularity.
    * Monitor and observe LLMs to better understand and evaluate performance, usage, and budgets.

    ## Products

    ### Traces

    W&B’s LLM tool is called *Traces*. **Traces** allow you to track and visualize the inputs and outputs, execution flow, model architecture, and any intermediate results of your LLM chains.

    ### Weave

    ### How it works

    ## Integrations
    =====
    Label: other
    Prediction: visualisation
    Query: Query: Weights & Biases dashboard features
     Retrieved Document: ## Tutorial
    ### Dashboards
    Now we can look at the results. The run we have executed is now shown on the left side, in our project, with the group and experiment names we listed. We have access to a lot of information that W&B has automatically recorded.

    We have several sections like:

    * Charts - contains information about losses, accuracy, etc. Also, it contains some examples from our data.
    * System - contains system load information: memory usage, CPU utilization, GPU temp, etc. This is very useful information because you can control the usage of your GPU and choose the optimal batch size.
    * Model - contains information about our model structure (graph).
    * Logs - include Keras default logging.
    * Files - contains all files that were created during the experiment, such as: config, best model, output logs, requirements, etc. The requirements file is very important because, in order to recreate a specific experiment, you need to install specific versions of the libraries.
    =====
    Label: other
    Prediction: visualisation
    Query: Query: W&B logging features
     Retrieved Document: ## Managing and Tracking ML Experiments With W&B
    ### Logging Advanced Things​

    One of the coolest things about W&B is that you can literally log anything. You can log custom metrics, matplotlib plots, datasets, embeddings from your models, prediction distribution, etc.

    Recently, Weights & Biases announced the Tables feature, which allows you to log, query and analyze tabular data. You can even visualize model predictions and compare them across models. For example: see the image below (taken from[ W&B Docs](http://docs.wandb.ai)), which compares two segmentation models.

    You can log audio data, images, histograms, text, video, and tabular data and visualize/inspect them interactively. To learn more about W&B Tables, go through their [documentation](https://docs.wandb.ai/guides/data-vis).

    You can even export the dashboard in CSV files to analyze them further. W&B also supports exports in PNG, SVG, PDF, and CSV, depending on the type of data you are trying to export.
    =====
    Label: other
    Prediction: artifact
    Query: Query: optimize W&B storage
     Retrieved Document: ## Storage

    If you are approaching or exceeding your storage limit, there are multiple paths forward to manage your data. The path that's best for you will depend on your account type and your current project setup.

    ## Manage storage consumption

    W&B offers different methods of optimizing your storage consumption:

    * Use reference artifacts to track files saved outside the W&B system, instead of uploading them to W&B storage.
    * Use an external cloud storage bucket for storage. *(Enterprise only)*

    ## Delete data

    You can also choose to delete data to remain under your storage limit. There are several ways to do this:

    * Delete data interactively with the app UI.
    * Set a TTL policy on Artifacts so they are automatically deleted.
    =====
    Label: other
    Prediction: visualisation
    Query: Query: experiment tracking
     Retrieved Document: ## Track Experiments
    ### How it works
    Track a machine learning experiment with a few lines of code:
    1. Create a W&B run.
    2. Store a dictionary of hyperparameters, such as learning rate or model type, into your configuration (`wandb.config`).
    3. Log metrics (`wandb.log()`) over time in a training loop, such as accuracy and loss.
    4. Save outputs of a run, like the model weights or a table of predictions.

    The proceeding pseudocode demonstrates a common W&B Experiment tracking workflow:

    ```python showLineNumbers

    # 1. Start a W&B Run

    wandb.init(entity="", project="my-project-name")

    # 2. Save mode inputs and hyperparameters

    wandb.config.learning\_rate = 0.01

    # Import model and data

    model, dataloader = get\_model(), get\_data()

    # Model training code goes here

    # 3. Log metrics over time to visualize performance

    wandb.log({"loss": loss})

    # 4. Log an artifact to W&B

    wandb.log\_artifact(model)
    ```
    =====

### Few Shot Examples

Building on our improved system prompt, we'll now add few-shot examples to our classifier. Few-shot examples provide concrete demonstrations of how to handle tricky edge cases, teaching the model through specific instances rather than abstract rules. This approach is particularly effective for resolving the context confusion and over-eagerness issues we identified in our error analysis.

For each label category, we've carefully selected examples that illustrate:

- Clear positive cases that should be assigned to that category
- Negative cases that might seem related but actually belong elsewhere

An example is when we show queries which were previously classified as integrations (Eg. using AWS IAM ) as others since these are authorization related.

```python
import instructor
from instructor_classify.schema import LabelDefinition, ClassificationDefinition, Examples
from openai import OpenAI

client = instructor.from_openai(OpenAI())

artifact_label = LabelDefinition(
    label="artifact",
    description="This is a user query and document pair which is about creating, versioning and managing weights and biases artifacts.",
    examples=Examples(
        examples_positive=[
            "How do I version a weights and biases artifact?",
        ],
        examples_negative=[
            "How do I do log an image with weights and biases?",
        ]
    )
)
integrations_label = LabelDefinition(
    label="integrations",
    description="this is a user query and document pair which is concerned with how we can integrate weights and biases with specific libraries. Note that Wandb is the weights and biases sdk's name so all questions about using it specifically will not be an integration question.",
    examples=Examples(
        examples_positive=[
            "How do I use weights and biases with keras?",
        ],
        examples_negative=[
            "how do I do distributed logging and tracing with weights and biases?",
            "what are some common issues that users face when using weights and biases?",
            "Does weights and bias support IAM access control?"
        ]
    )
)

visualisation_label = LabelDefinition(
    label="visualisation",
    description="This is a user query and document pair which is concerned about how we can visualise the data that we've logged with weights and biases",
    examples=Examples(
        examples_positive=[
            "How do I visualise my training runs?",
        ],
        examples_negative=[
            "Does Weights and biases support AWS IAM authentication when using SageMaker?",
        ]
    )
)

other_label = LabelDefinition(
    label="other",
    description="Use this label for other query types which don't belong to any of the other defined categories that you have been provided with",
    examples= Examples(
        examples_positive=[
            "How do I do a hyper-parameter search with weights and biases?",
            "How do I log an image with weights and biases?",
            "Can I deploy weights and biases in my own infrastructure?",
            "Does weights and bias support IAM access control with sagemaker?",
            "How do I initialise weights and biases?",
        ],
        examples_negative=[
            "How do I save a weights and biases artifact?",
        ]
    )

)


classification_def_w_system_prompt_and_examples = ClassificationDefinition(
    system_message="""
You are a world class classifier. You are given a user query and a document.

Look closely at the user query and determine what the user's query is about. You will also be provided with a document which is relevant to the user's query. It might contain additional information that's not relevant to the user's query so when deciding on the category, only consider the parts that are relevant to answering the user's specific question.

Here are the categories you can choose from:

1. Artifacts - questions about creating, versioning and managing weights and biases artifacts. Note that in W&B there are two ways to store data - Artifacts and Files. Only consider queries about artifacts when they explicitly mention the usage of Artifacts with W&B.
2. Integrations - questions about integrating weights and biases with specific libraries. This should only be used for questions about using the weights and biases sdk with specific libraries.
3. Visualisation - questions about visualising the data that we've logged with weights and biases
4. Other - queries that don't fit the above categories.

Note that in weights and biases there are two ways to store data - Artifacts and Files. Only consider queries about artifacts when they explicitly mention the usage of Artifacts with W&B. Just logging data isn't sufficient to be classified as an artifact, it must explicitly reference or use the Artifact API.

Choose a single category for the user query and document pair that's the best match""",
    label_definitions=[artifact_label, other_label, visualisation_label, integrations_label],
)

```

```python
classifier_v3 = (
    Classifier(classification_def_w_system_prompt_and_examples).with_client(client).with_model("gpt-4.1-mini")
)
predictions_system_prompt_and_examples = predict_and_evaluate(classifier_v3, val_text, val_labels)
predictions_system_prompt_and_examples["accuracy"]

```

    0.9090909090909091

```python
for prediction,label,query in zip(predictions_system_prompt_and_examples["predictions"],predictions_system_prompt_and_examples["labels"],predictions_system_prompt_and_examples["queries"]):
    if label != prediction:
        print(f"Label: {label}")
        print(f"Prediction: {prediction}")
        print("## Query")
        print(f"{query}")
        print("=====")
```

    Label: other
    Prediction: integrations
    ## Query
    Query: Are there any best practices for using wandb in a distributed training environment?
     Retrieved Document: ## Add wandb to Any Library
    #### Distributed Training

    For frameworks supporting distributed environments, you can adapt any of the following workflows:

    * Detect which is the “main” process and only use `wandb` there. Any required data coming from other processes must be routed to the main process first. (This workflow is encouraged).
    * Call `wandb` in every process and auto-group them by giving them all the same unique `group` name

    See Log Distributed Training Experiments for more details
    =====
    Label: other
    Prediction: visualisation
    ## Query
    Query: Weights & Biases features for LLM developers
     Retrieved Document: **Weights & Biases Prompts** is a suite of LLMOps tools built for the development of LLM-powered applications.

    Use W&B Prompts to visualize and inspect the execution flow of your LLMs, analyze the inputs and outputs of your LLMs, view the intermediate results and securely store and manage your prompts and LLM chain configurations.

    #### 🪄 View Prompts In Action

    **In this notebook we will demostrate W&B Prompts:**

    * Using our 1-line LangChain integration
    * Using our Trace class when building your own LLM Pipelines

    See here for the full W&B Prompts documentation

    ## Installation

    ```
    !pip install "wandb>=0.15.4" -qqq
    !pip install "langchain>=0.0.218" openai -qqq

    ```

    ```
    import langchain
    assert langchain.__version__ >= "0.0.218", "Please ensure you are using LangChain v0.0.188 or higher"

    ```

    ## Setup

    This demo requires that you have an OpenAI key

    # W&B Prompts

    W&B Prompts consists of three main components:

    **Trace table**: Overview of the inputs and outputs of a chain.
    =====
    Label: other
    Prediction: artifact
    ## Query
    Query: log prompts wandb
     Retrieved Document: def log_index(vector_store_dir: str, run: "wandb.run"):
        """Log a vector store to wandb

        Args:
            vector_store_dir (str): The directory containing the vector store to log
            run (wandb.run): The wandb run to log the artifact to.
        """
        index_artifact = wandb.Artifact(name="vector_store", type="search_index")
        index_artifact.add_dir(vector_store_dir)
        run.log_artifact(index_artifact)

    def log_prompt(prompt: dict, run: "wandb.run"):
        """Log a prompt to wandb

        Args:
            prompt (str): The prompt to log
            run (wandb.run): The wandb run to log the artifact to.
        """
        prompt_artifact = wandb.Artifact(name="chat_prompt", type="prompt")
        with prompt_artifact.new_file("prompt.json") as f:
            f.write(json.dumps(prompt))
        run.log_artifact(prompt_artifact)
    =====
    Label: other
    Prediction: visualisation
    ## Query
    Query: Weights & Biases GPU utilization
     Retrieved Document: ## Monitor & Improve GPU Usage for Model Training
    #### 1. Measure your GPU usage consistently over your entire training runs
    You can’t improve GPU usage without measuring it.  It’s not hard to take a snapshot of your usage with useful tools like nvidia-smi, but a simple way to find issues is to track usage over time.  Anyone can turn on system monitoring in the background, which will track GPU, CPU, memory usage etc over time by adding two lines to their code:

    ```
    import wandb
    wandb.init()
    ```

    The wandb.init() function will create a lightweight child process that will collect system metrics and send them to a wandb server where you can look at them and compare across runs with graphs like these:

    The danger of taking a single measurement is that GPU usage can change over time.  This is a common pattern we see where our user Boris is training an RNN; mid-training, his usage plummets from 80 percent to around 25 percent.
    =====
    Label: other
    Prediction: visualisation
    ## Query
    Query: Weights & Biases dashboard features
     Retrieved Document: ## Tutorial
    ### Dashboards
    Now we can look at the results. The run we have executed is now shown on the left side, in our project, with the group and experiment names we listed. We have access to a lot of information that W&B has automatically recorded.

    We have several sections like:

    * Charts - contains information about losses, accuracy, etc. Also, it contains some examples from our data.
    * System - contains system load information: memory usage, CPU utilization, GPU temp, etc. This is very useful information because you can control the usage of your GPU and choose the optimal batch size.
    * Model - contains information about our model structure (graph).
    * Logs - include Keras default logging.
    * Files - contains all files that were created during the experiment, such as: config, best model, output logs, requirements, etc. The requirements file is very important because, in order to recreate a specific experiment, you need to install specific versions of the libraries.
    =====
    Label: other
    Prediction: visualisation
    ## Query
    Query: W&B logging features
     Retrieved Document: ## Managing and Tracking ML Experiments With W&B
    ### Logging Advanced Things​

    One of the coolest things about W&B is that you can literally log anything. You can log custom metrics, matplotlib plots, datasets, embeddings from your models, prediction distribution, etc.

    Recently, Weights & Biases announced the Tables feature, which allows you to log, query and analyze tabular data. You can even visualize model predictions and compare them across models. For example: see the image below (taken from[ W&B Docs](http://docs.wandb.ai)), which compares two segmentation models.

    You can log audio data, images, histograms, text, video, and tabular data and visualize/inspect them interactively. To learn more about W&B Tables, go through their [documentation](https://docs.wandb.ai/guides/data-vis).

    You can even export the dashboard in CSV files to analyze them further. W&B also supports exports in PNG, SVG, PDF, and CSV, depending on the type of data you are trying to export.
    =====
    Label: other
    Prediction: artifact
    ## Query
    Query: optimize W&B storage
     Retrieved Document: ## Storage

    If you are approaching or exceeding your storage limit, there are multiple paths forward to manage your data. The path that's best for you will depend on your account type and your current project setup.

    ## Manage storage consumption

    W&B offers different methods of optimizing your storage consumption:

    * Use reference artifacts to track files saved outside the W&B system, instead of uploading them to W&B storage.
    * Use an external cloud storage bucket for storage. *(Enterprise only)*

    ## Delete data

    You can also choose to delete data to remain under your storage limit. There are several ways to do this:

    * Delete data interactively with the app UI.
    * Set a TTL policy on Artifacts so they are automatically deleted.
    =====

```python
import matplotlib.pyplot as plt
from sklearn.metrics import ConfusionMatrixDisplay

# Get unique labels
unique_labels = ["artifact", "other", "visualisation", "integrations"]

# Convert predictions and true labels to label indices
y_true = [unique_labels.index(label) for label in predictions_system_prompt_and_examples["labels"]]
y_pred = [unique_labels.index(label) for label in predictions_system_prompt_and_examples["predictions"]]

# Calculate single confusion matrix for all categories
conf_matrix = confusion_matrix(y_true, y_pred)

# Plot confusion matrix
plt.figure(figsize=(10, 8))
disp = ConfusionMatrixDisplay(confusion_matrix=conf_matrix, display_labels=unique_labels)
disp.plot()
plt.title('Confusion Matrix for All Categories')
plt.tight_layout()
plt.show()
```

    <Figure size 1000x800 with 0 Axes>

![png](extra_kura_03_classifiers_files/extra_kura_03_classifiers_37_1.png)

Let's now see the performance of this classifier on the test set

```python
predictions_system_prompt_and_examples = predict_and_evaluate(classifier_v3, test_text, test_labels)
predictions_system_prompt_and_examples["accuracy"]

```

    0.9090909090909091

# Performance Evolution: From Baseline to Optimized Classifier

Let's examine how systematic changes to our prompting strategy transformed our classifier's performance:

| Prompt   | Baseline | System Only    | System + Examples |
| -------- | -------- | -------------- | ----------------- |
| Accuracy | 72.7%    | 80.5% (+10.7%) | 90.9% (+25.0%)    |

These improvements demonstrate the power of thoughtful prompt engineering. By adding a clear system prompt, we saw a significant 10.7% relative improvement. When we further enhanced this with carefully selected examples, we achieved an additional 25% gain relative to our baseline, bringing our final validation accuracy to 90.9%.

The real test came with our holdout test set, where we maintained this high performance level - achieving 90.9% accuracy compared to the baseline's 69.7%. This consistency between validation and test performance suggests our improvements are robust and generalizable.

## Application

Now that we've built and validated a classifier with over 90% accuracy, we can confidently apply it to our entire dataset to understand the true distribution of user queries. This isn't just an academic exercise - it's a powerful tool for product development

```python
with open("./data/conversations.json") as f:
    conversations_full = json.load(f)

dataset_texts = [
   f"Query:{item['query']}\nRetrieved Document:{item['matching_document']}" for item in conversations_full
]
```

```python
dataset_labels = classifier_v3.batch_predict(dataset_texts)
```

```python
from collections import Counter

Counter([item.label for item in dataset_labels])
```

    Counter({'other': 285,
             'artifact': 115,
             'integrations': 83,
             'visualisation': 77})

## Conclusion

### What You Learned

In this final notebook, you learned how to bridge the gap between discovery and production by building robust query classifiers. You discovered how to:

- **Generate weak labels efficiently** using automated classification and human review workflows
- **Build production classifiers** using the `instructor` library with systematic prompt engineering
- **Iteratively improve performance** through enhanced system prompts and few-shot examples
- **Evaluate and measure accuracy** using confusion matrices and validation/test splits
- **Apply classifiers at scale** to understand true query distributions across your dataset

### What We Accomplished

We built a production-ready classifier that achieved 90.9% accuracy through systematic prompt engineering, starting from a 72.7% baseline. Our iterative improvements—adding clear system prompts (+10.7%) and strategic few-shot examples (+25% total)—demonstrated the power of thoughtful prompt design.

Applying our classifier to the full dataset revealed that just three categories—artifacts (20%), integrations (15%), and visualizations (14%)—account for roughly 50% of all user conversations. This concentration is significant because it means we can focus improvement efforts on these specific areas to impact half of all user interactions with measurable precision:

- **Artifacts**: Analyze specific operations users struggle with (versioning, linking, metadata management) to build specialized guides and automated workflows
- **Integrations**: Identify which libraries generate the most questions to create framework-specific tutorials and testing tools
- **Visualizations**: Understand common visualization needs to build specialized UIs and simpler workflows

More importantly, this systematic approach transforms vague user feedback into actionable product insights, enabling engineering teams to prioritize features, documentation teams to focus efforts, and product teams to make data-driven roadmap decisions.

### Beyond Classification: The Production Frontier

With our classifier successfully identifying query patterns at 90%+ accuracy, you've built the foundation for advanced RAG improvements that extend far beyond this tutorial series:

- **Specialized Retrieval Pipelines**: Build dedicated embedding models and retrieval strategies optimized for each category
- **Proactive Monitoring**: Detect when artifact queries surge, signaling potential documentation gaps or product issues
- **Intelligent Query Routing**: Automatically direct integration questions to framework-specific knowledge bases or expert systems
- **Closed-loop Feedback**: Measure which query types have the lowest satisfaction scores to prioritize improvements

The key insight from this entire series is that improving RAG systems isn't about better models or more data—it's about understanding your users systematically and building focused solutions for their actual needs. This methodology can be applied to any domain where you have query-document pairs and want to move from reactive fixes to proactive, data-driven improvements.

---

IF you want to get discounts and 6 day email source on the topic make sure to subscribe to

[Enroll in the Free 6-Day Email Course](https://improvingrag.com/){ .md-button .md-button--primary }

---

IF you want to get discounts and 6 day email source on the topic make sure to subscribe to

[Enroll in the Free 6-Day Email Course](https://improvingrag.com/){ .md-button .md-button--primary }

---

IF you want to get discounts and 6 day email source on the topic make sure to subscribe to

[Enroll in the Free 6-Day Email Course](https://improvingrag.com/){ .md-button .md-button--primary }

---

IF you want to get discounts and 6 day email source on the topic make sure to subscribe to

[Enroll in the Free 6-Day Email Course](https://improvingrag.com/){ .md-button .md-button--primary }

---

IF you want to get discounts and 6 day email source on the topic make sure to subscribe to

<script async data-uid="010fd9b52b" src="https://fivesixseven.kit.com/010fd9b52b/index.js"></script>
