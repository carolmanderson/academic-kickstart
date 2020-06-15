---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Extracting Food Mentions From Recipes"
summary: 
authors: []
tags: []
categories: []
date: 2020-05-03T16:46:14-06:00

# Optional external URL for project (replaces project detail page).
external_link: ""

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Custom links (optional).
#   Uncomment and edit lines below to show custom links.
# links:
# - name: Follow
#   url: https://twitter.com
#   icon_pack: fab
#   icon: twitter

url_code: ""
url_pdf: ""
url_slides: ""
url_video: ""

# Slides (optional).
#   Associate this project with Markdown slides.
#   Simply enter your slide deck's filename without extension.
#   E.g. `slides = "example-slides"` references `content/slides/example-slides.md`.
#   Otherwise, set `slides = ""`.
slides: ""
---
As a toy Named Entity Recognition project, I trained a model to extract food mentions from recipes. To play with the trained model, see the web app [here](http://54.213.148.85:8501). I built the app using the python library Streamlit, and deployed it on AWS.




## Training Data
In my experience, getting labeled data is the most time-consuming and difficult part of supervised learning projects. As my training data, I used [this dataset from Kaggle](https://www.kaggle.com/hugodarwood/epirecipes), consisting of more than 20,000 recipes from [Epicurious.com](www.epicurious.com). I hand-labeled food mentions in a random subset of 300 recipes, consisting of 981 sentences, using the awesome labeling tool [Prodigy](https://prodi.gy). 


### Exploring the data
The Kaggle dataset includes many fields besides the recipes themselves, such as Title, Date, Category, Calories, Fat, Protein, Sodium, and Ingredients. In future projects, I'll use these as labels to train classification or regression models. For now, I delved into two questions:

* What kinds of foods are most commonly included in these recipes?
* Are there any interesting patterns in the nutrition information?

#### Common foods
To answer the first question, I determined the most common foods after labeling 300 recipes. The most common one-word foods were:

| Term      | # Mentions|
| ----------| ----------|
| salt    | 301       |
| pepper    | 202       |
| oil       | 122       |
| butter    | 122       |
| sugar     | 114       |
| sauce     | 89       |
| garlic    | 78       |
| dough    | 69       |
| onion    | 66       |
| flour    | 63       |


And the most common two-word food terms were:

| Term      | # Mentions|
| ----------| ----------|
|lemon juice| 47
|olive oil|29
|lime juice| 16
|flour mixture| 12
|sesame seeds| 11
|egg mixture| 10
|pan juices|10
|baking powder| 10
|cream cheese|10
|green onions| 10  


You can see that the recipes include both baking and non-baking cooking. They span a wide variety of cooking styles and techniques. They also include cocktail recipes and a detailed set of instructions for setting up a beachside clambake! 

#### Nutrition info
Each recipe includes nutrition information, but the information is hard to interpret. What are the units? Are the quantities per serving or for the whole recipe? For examplel the Lamb Rice Pilaf recipe lists a sodium quantity of 3,134,853. If we assume the units are mg, this is 3 kilograms of sodium, an amount that most people don't have in their pantry.

After removing unreasonable quantities of sodium, we can see a more reasonable distribution:

![Sodium Plot](/img/sodium.jpg)

The other nutritional values follow a similar pattern — some crazy-high outliers, and a reasonable distribution if outliers are removed. 


### Difficult labeling decisions
You'd think labeling food would be easy, but I faced some hard decisions. Is water food? Sometimes water is an ingredient, but other times it's used almost as a tool, as in "plunge asparagus into boiling water." What about phrases like "egg mixture"? Should "mixture" be considered part of the food item? Or how about the word "slices" in the sentence "Arrange slices on a platter"? These decisions would normally be guided by  a business use case. But here I was just training a model for fun, so these decisions were tough, and a bit arbitrary. 

## Model Architecture
As a starting point, I used a simple model with a bidirectional LSTM layer followed by a softmax output layer to predict token labels. I used pretrained GloVe embeddings as the only feature. I trained the model on Google Colab; you can see the training code [here.](https://github.com/carolmanderson/food/blob/master/notebooks/modeling/Train_basic_LSTM_model.ipynb)

## Model Performance
The best checkpoint achieved scores on the dev set of both precision and recall of 92% when evaluated at the entity level (i.e., the exact beginning and end of each `FOOD` entity must match the ground truth labels in order to count as a true postive). 

The best dev set F1 score was achieved after five training epochs with a batch size of four; I allowed model training to continue until ten epochs had occurred without a score improvement, for a total of 15 epochs. 


## Next steps
Even though this model is simple, it works pretty well. Error analysis shows that the most common errors it makes are `BIO` errors. To understand what these are, you need to know that the tags the model predicts are `B-FOOD`, `I-FOOD`, and `O`. `B-FOOD` indicates a token that is the first token in a food mention (B=beginning). `I-FOOD` is a second or later token in a food mention (I=inside). `O` is a token that isn't part of a food mention (O=outside). The model's most common error type is predicting `I-FOOD` for a token that should be `B-FOOD`, or vice-versa. A conditional random field (CRF) layer would probably help reduce this type of error, since it predicts the most probable sequence of labels. A CRF can learn, for example, that I-FOOD should never occur immediately after O.

Another error type involves words with multiple meanings. The word "salt", for example, can be a food, but it can also be a verb, as in "Lightly salt the vegetables." The word "butter" is similarly ambiguous — you can use butter as a ingredient, but you can also "butter the pan." The current model seems to treat all mentions of these words as food:


![Butter and Salt](/img/butter_salt_v3.png)

These kinds of errors could probably be reduced by using contextual word embeddings, such as ELMo or BERT embeddings, or by using BERT itself (fine-tuning it on the NER task). 

#### Try the live demo [here.](http://54.213.148.85:8501)  

