---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Classifying Recipes for Special Diets"
summary: 
authors: []
tags: []
categories: []
date: 2020-06-11T16:46:14-06:00

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
I mostly eat vegan. I used to be fully vegan, but now I allow myself some exceptions. If there were a vegan license, mine would be revoked. But I'm still always looking for new vegan recipes. In this project, I built a model to classify recipes as vegan or not. While I was at it, I also trained models to classify gluten-free and kosher recipes. You can play around with a live demo of the models [here.](http://54.213.148.85:8501)

 
## Training Data
I used [this dataset from Kaggle](https://www.kaggle.com/hugodarwood/epirecipes), consisting of more than 20,000 recipes from [Epicurious.com](www.epicurious.com). Recipes in the data set are assigned numerous categories, which include "gluten-free", "kosher", and "vegan."

### Exploring the data
Out of the 20,000 recipes in the full data set, 5511 are kosher, 4357 are gluten-free, and 1535 are vegan. This represents a significant class imbalance, which is a potential problem: models trained and tested on imbalanced data can achieve high accuracy, but low predictive power, by tending to predict an example belongs to the majority class. For example, since fewer than 10% of the recipes are vegan, a model that simply predicts every recipe is non-vegan would achieve 90% accuracy. I used two strategies to address this class imbalance. First, I downsampled the data to reduce the number of non-gluten-free, non-kosher, and non-vegan recipes. Second, I experimented with weighted loss, placing more weight on errors involving the minority classes. In my downsampled data set, the proportions of each class were:

| Class      | # of recipes |% of recipes |
| ----------| ----------|----------|
| Gluten Free    |   4357     |40
| Kosher    |   5511     |50
| Vegan       |   1535     |14


Note that these appear to add up to more than 100% because a recipe can be in multiple categories. The total number of recipes was 11,022.


#### Common ingredients
In a previous project, I trained a model to extract food mentions from recipes. I used this model to find out which ingredients were most over- or under-represented in each class of recipe. 

#### Gluten-free
| Under-represented      | Over-represented|
| ----------| ----------|
| breadcrumbs    | potatoes       |
| soy    | white pith       |
| toasts       | gelatin       |
| crumbs    | vinaigrette       |
| flour     | beets       |
| meal     | asparagus       |
| pasta    | squash       |
| cocktail    | turmeric       |
| tart    | radishes       |
| soda    | chile       |

#### Kosher
| Under-represented      | Over-represented|
| ----------| ----------|
| bacon    | frosting       |
| shrimp    | cookies       |
| scallops       | blueberries       |
| gelatin    | powdered sugar       |
| pork     | oats       |
| ham     | vanilla       |
| cocktail    | pudding       |
| sausage    | cake       |
| chops    | meringue       |
| drippings    | ice cream       |


#### Vegan
| Under-represented      | Over-represented|
| ----------| ----------|
| scallops    | white pith       |
| yolks    | eggplant       |
| yolk       | broccoli       |
| bacon    | sesame       |
| chops     | vinaigrette       |
| beef     | oranges       |
| breast    | sesame seeds       |
| buttermilk    | zucchini       |
| salmon    | beets       |
| egg yolk    | turmeric      |

There are some interesting patterns here. The under-represented ingredients fit with my expectations for each category, and this gives me some confidence in the labels. The over-represented ingredients seem less informative. It's surprising to find that `white pith`, `vinaigrette`, `beets`, and `turmeric` are overrepresented in both the vegan and gluten-free classes, since they're not very common terms in general. My guess is that those ingredients are unusually common in this particular recipe collection.

It was interesting to find `cocktail` in the kosher under-represented terms; I wondered whether it came from shrimp cocktail. A deeper dive revealed that the vast majority of `cocktail` mentions in non-kosher recipes were, in fact, adult beverages. 

I was also intrigued by `soy` in the list of gluten-free under-represented terms. This turned out to arise from `soy sauce`, which contains gluten. The food NER model was trained to recognize entire terms, and thus should have found `soy sauce` rather than `soy`, but it appears not to have learned that term.

## Model Architecture
I was originally planning to use a CNN-based model, with word embeddings as features, to classify the recipes. However, since I had already extracted food terms from the recipes, I decided instead to try using those terms as features in non-deep-learning models. I tried several different model types: logistic regression, naive bayes, random forest, multilayer perceptron, and XGBoost. For each architecture, I trained a separate binary classifier for each label (gluten-free, kosher, and vegan), and tuned hyperparameters for each task. For all three labels, the best-performing model was XGBoost. I also experimented with different ways of representing the food term frequencies: simple counts, TF-IDF, or using only the presence or absence of each term in a recipe and ignoring how many times it appears in the recipe. The best-performing XGBoost models used simple counts.  


### Experimenting with weighted loss  
Because the gluten-free and vegan categories were still imbalanced after downsampling the data, I experimented with using class weights to improve performance on those classes. This approach places more emphasis during training on errors involving the minority classes. The results below show an experiment on the vegan-or-not classifier. Applying class weights improves overall performance on the vegan class by enhancing recall at the expense of precision. The converse is true for the non-vegan class: precision is improved at the expense of recall. Since I wanted the classifier to capture as many true vegan recipes as possible, I used the model trained with weighted loss. 

##### Unweighted
|  Label     | Precision| Recall | F1-Score |Support |
| ----------| ----------|----------|----------|----------|
|`Non-vegan`|0.94|0.96|0.95|1896|
|`Vegan`    |0.69|0.60|0.64| 308|
|accuracy|     |    |0.91|2204|
|macro avg|0.81|0.78|0.79|2204|
|weighted avg|0.90|0.91|0.90|2204|

##### Weighted
|  Label     | Precision| Recall | F1-Score |Support |
| ----------| ----------|----------|----------|----------|
|`Non-vegan`|0.98|0.89|0.93|1896|
|`Vegan`    |0.57|0.87|0.69| 308|
|accuracy|     |    |0.89|2204|
|macro avg|0.77|0.88|0.81|2204|
|weighted avg|0.92|0.89|0.90|2204|


## Model Performance
After model selection, feature selection, and hyperparameter tuning, I trained three final XGBoost models on the combined training and dev sets (one model each for gluten-free, kosher, and vegan recipes). The performance of each model on the test set is shown below.

|  Label     | Precision| Recall | F1-Score |Support |
| ----------| ----------|----------|----------|----------|
|`Non-gluten-free`|0.70|0.88|0.78|1064|
|`Gluten-free`    |0.85|0.65|0.73|1141|
|`Non-kosher`|0.68|0.78|0.72| 954|
|`Kosher`    |0.81|0.71|0.76|1251|
|`Non-vegan`|0.91|0.98|0.95|1767|
|`Vegan`    |0.89|0.62|0.73| 438|

## Error analysis
I found that the majority of the false positives were due to labeling errors, i.e. the recipe publisher failing to mark a recipe as vegan, gluten-free, or kosher when it actually should have been marked as such. There were some model errors, too, though. For example, a pasta dish containing `farfalle` was predicted to be gluten-free. This word was never seen during training, so the model could not have learned that it isn't gluten-free. Another recipe containing a `ham hock` was incorrectly predicted to be vegan. The features I used were entire terms (`ham hock` rather than `ham` and `hock`), and `ham hock` was not found in the training set; so again, the model had no chance to learn that it wasn't vegan. The latter problem could be addressed by tokenizing terms and using individual tokens; however, some information would be lost that way. For example, `rice flour` is gluten-free, but other kinds of flour aren't. 

Most of the false negatives appeared to be errors in the predictions rather than in the labels. I didn't find any clear pattern to the false negatives. We can't blame this on imbalanced data, because even the kosher-or-not model, whose data set was perfectly balanced, suffered from low recall. My guess is that the underlying cause is that the labels were inconsistent, making the task difficult to learn. For example, two similar recipes containing the same ingredients might be marked `vegan` and `non-vegan`, respectively, based on who wrote or published the recipe.  


## Next steps
Based on the error analysis above, I think a CNN—based model using word embeddings might perform better than the current one. Word embeddings would help the model know that `farfalle` is a type of pasta, while the CNN would capture information from multi-word phrases such as `rice flour`. A BERT-based model would be another reasonable option. Given the problems I found in the labels, I would also consider taking steps to clean up the training data if I wanted to get serious about improving this model. 


#### Try the live demo [here.](http://54.213.148.85:8501)  

