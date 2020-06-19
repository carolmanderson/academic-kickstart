---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Smart Document Batching"
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

To train deep learning models fast, we pack multiple training examples into mini-batches. But how to deal with documents of variable length when training on text?

The answer is padding: we pad all documents in a batch, usually with zeroes, to the length of the longest document in the batch. But this can, paradoxically, _slow down_ training, because it can turn short documents into long ones.

## Smart Batching to the Rescue

To make mini-batch training faster, I wanted to group documents of similar size together, thereby reducing the average amount of padding per document. 

But here's the tricky part: I didn't want the same training examples to be batched together in every epoch. The easiest approach would have been to sort the documents by length, and then create batches by iterating over the sorted list. But in that case we would have had the _exact same batches_ in every epoch. My intuition (though I haven't tested this) is that that approach might cause the network to overfit to the specific mini-batches seen during training, or to the specific amount of padding on particular documents. It would be better to vary the batches so that the network sees different combinations of documents, and different amounts of padding on each individual document.

I wrote the code below to group similar-length documents together, but in a variable way that ensures the batches are different in each epoch. 


## The Algorithm
Here's the logic behind the algorithm. Skip this part if you just want to see the results or the code!

1. Make a list containing the length of each document, in tokens, at the corresponding index in the training set.
2. Randomly choose a starting index. Add the document at that index to the current batch.
3. If there are any documents of the same length as the first document, add them to the batch. Stop if the desired batch size is reached.
4. If the batch isn't full, randomly pick a direction (up or down, towards shorter or longer documents).
5. If the chosen direction is up, look at longer documents and find the document closest in length to the documents already in the batch. Add it to the batch. Continue adding longer documents until the desired batch size is reached or no unused longer documents are available. If the batch is still not full, switch directions and look for smaller documents. (If the chosen direction is down, go through a similar procedure, but start with shorter documents.)
6. Once each batch is full, initiate the next batch by choosing another random index. Continue until all training examples have been packed into batches.


## The Results 

The figure below shows a simple example — I used 32 documents sampled from a real data set, and packed them into minibatches of 4. The bars represent individual documents, and each panel shows how the documents were batched in a particular epoch. Documents of the same color were batched together. As you can see, the minibatches tended to contain documents of similar length, but didn't always include exactly the same documents — just as I wanted. 

![Batches](/img/all_epochs.jpg)


## Does This Actually Help Speed Up Training?

Yes it does! In an experiment with a real training data set, using a batch size of 16, I found that my training time per epoch was 61 seconds with smart batching, compared to 115 seconds with random batching. It almost cut training time in half! The speed benefit will, of course, vary depending on the distribution of document lengths in the data set and the minibatch size. The benefit is greatest when the data set contains a mixture of very short and very long documents.

## The code: 

<script src="https://gist.github.com/carolmanderson/df37ae58c58231e74bffee6571b26f97.js"></script>
