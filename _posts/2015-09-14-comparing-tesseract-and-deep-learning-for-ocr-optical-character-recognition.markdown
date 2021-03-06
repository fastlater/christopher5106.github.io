---
layout: post
title:  "Compare Tesseract and deep learning techniques for Optical Character Recognition of license plates"
date:   2015-09-14 23:00:51
categories: computer vision
---

In this article, I follow techniques used in [Google Translate app](http://googleresearch.blogspot.fr/2015/07/how-google-translate-squeezes-deep.html) for the case of license plates and I compare performances of deep learning nets with what we could have previously done with Tesseract engine.

![Cat]({{ site.url }}/img/plate_recognition2.jpg)

I first created a simple "plate annotation tool"

    ./annotate input_dir output.csv

in order to create a text file labelling the data, one line per character

    image_path,character,x,y,width,height,orientation

in a CSV format. License plates are detected with a cascade classifier and letters with the findContours method from OpenCV. I filter contours that are too small compared to the license plate dimensions, and contours that do not contain enough black/dark colors, because french license plates are always written in black color.

I created a conversion tool

    ./extract file.csv output_dir --backend=[lmdb|leveldb|directory|tesseract]

to convert this CSV file to their respective format for [Tesseract]({{ site.url }}/optical/character/recognition/2015/09/01/training-optical-character-recognition-technology-tesseract.html) and [Caffe learning]({{ site.url }}/deep/learning/2015/09/04/Deep-learning-tutorial-on-Caffe-Technology.html). The *directory* option exports the extracts into a tree of directories, one directory per letter, which is a great way for me to check the labelling errors.

For Tesseract, this will bring me such a Tiff file with a .box file :

![Tiff file for Tessearct]({{ site.url }}/img/lpfra.std.exp0.jpg)

For Caffe, it will populate a LMDB database that I can inspect in Python :

{% highlight python %}
import caffe
import numpy as np
import lmdb
import matplotlib.pyplot as plt

env = lmdb.open('test_lmdb')
t = ['A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R','S','T','U','V','W','X','Y','Z','0','1','2','3','4','5','6','7','8','9'];

def get(k):
    with env.begin() as txn:
        raw_datum = txn.get(k)

    datum = caffe.proto.caffe_pb2.Datum()
    datum.ParseFromString(raw_datum)
    flat_x = np.fromstring(datum.data, dtype=np.uint8)
    x = flat_x.reshape(datum.channels, datum.height, datum.width)
    y = datum.label
    plt.imshow(x[0,...],cmap='gray')
    plt.show()
    print t[y]

get(b'00000006')
get(b'00000009')
{% endhighlight %}

The training set is composed of 5000 letters, and the test set of 160 letters. For Tesseract I had to use a subset for the training set of 800 letters, otherwise training was not working properly.

For deep learning, I used a standard LeNet neural network with dropout layers.

I trained both technologies and here is the result :

| Technology        | Correct results           |
| ------------- |:-------------:|
| Tesseract eng language      | 64 |
| Tesseract trained language      | 71      |
| Caffe trained (NN)  | 154 |

For Tesseract I consider a match when the letter returned by the best confidence matches the ground truth.

Caffe is 97% right. The wrong matches are :

    M W
    0 D
    B 8
    1 A
    D 0
    D Q

Given that we can now infer the letter/number shema for a licence plate (LL-NNN-LL or NN-LLL-NN) with a good precision, it's in fact a **99% correctness**, that means *one wrong letter every hundred letters* thanks to deep learning.

There remains a group of letters that is still hardly distinguishable (O-D-Q) under poor conditions.

Let's try some optimizations : previous results were computed on the gray image of the image. Let's give a try on the thresholded image with

    cvtColor(image,gray_image,CV_BGR2GRAY);
    adaptiveThreshold(gray_image,threshold_image,255,ADAPTIVE_THRESH_MEAN_C,THRESH_BINARY,11,12);

The training file has letters with a better background :

![Tiff file for Tesseract]({{ site.url }}/img/lpfra.std.exp1.png)

| Technology        | Correct results           |
| ------------- |:-------------:|
| Tesseract eng language      | 82 |
| Tesseract trained language      | 93      |
| Caffe trained (NN)  | 152 |

Tesseract results are better on black & white images. Caffe results do not improve, this comes from the fact the first layers of the neural network do the preprocessing.
