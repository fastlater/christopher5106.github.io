---
layout: post
title:  "IBM Watson Bluemix Visual API : tutorial to create a custom classifier"
date:   2016-12-23 17:00:51
categories: computer vision
---

In this tutorial, I'll see how to use the standard visual classifier using IBM labels, then create our new custom classifier with Watson Visual API.


# Configure your Watson service

Go to the [Watson dashboard](https://console.ng.bluemix.net/dashboard/watson) in your Bluemix account, click on "Create a Watson service" and choose **Visual Recognition**, free usage. In the identification data for the service, create new data, which will return **url** and **api_key** values.

Let's set the `$API_KEY` shell variable to your IBM API KEY.

# Prepare Dataset

Let's download the [Food 101 dataset](https://www.vision.ee.ethz.ch/datasets_extra/food-101/) for our classification tutorial, to classify Food.

```bash
wget http://data.vision.ee.ethz.ch/cvl/food-101.tar.gz
tar xvzf food-101.tar.gz
```

Let's create the dataset :

- First, let's resize all images to 320 max dimension because IBM Watson Visual Recognition accepts images not more than 320x320 :

```bash
for file in food-101/images/*; do
  mogrify "$file/*.jpg[!320x320>]"
done
```

- Second, create train and test directories :

```bash
mkdir train test
for SPLIT in train test ;
do
  while read -r line
  do
      name="$line"
      DIRNAME="train/"`dirname $name`
      if [ ! -d "$DIRNAME" ]; then
        echo "mkdir $DIRNAME"
        mkdir $DIRNAME
      fi
      cp "food-101/images/$line.jpg" "$PLIT/$line.jpg"
      echo "Name read from file - $name"
  done < food-101/meta/$SPLIT.txt
done
```

- Last, zip train images to prepare upload to Watson :

```bash
for file in train/*; do
  echo "zipping $file" ; zip -r "$file.zip" "$file";
done
```

# Use the standard classifier

The standard classifier comes with a huge dictionary of labels, for which models have been trained by IBM. let's give a try on one of our test images :  

    curl -X POST -F "images_file=@test/apple_pie/1011328.jpg" "https://gateway-a.watsonplatform.net/visual-recognition/api/v3/classify?api_key=$API_KEY&version=2016-05-20"


![](img/apple_pie.jpg)

The classes retrieved for our apple pie are : sea hare, "shellfish, invertebrate, animal, giant conch...

    {
        "custom_classes": 0,
        "images": [
            {
                "classifiers": [
                    {
                        "classes": [
                            {
                                "class": "sea hare",
                                "score": 0.587,
                                "type_hierarchy": "/animal/invertebrate/shellfish  /sea hare"
                            },
                            {
                                "class": "shellfish  ",
                                "score": 0.919
                            },
                            {
                                "class": "invertebrate",
                                "score": 0.919
                            },
                            {
                                "class": "animal",
                                "score": 0.919
                            },
                            {
                                "class": "giant conch",
                                "score": 0.585,
                                "type_hierarchy": "/animal/invertebrate/shellfish  /giant conch"
                            },
                            {
                                "class": "conch",
                                "score": 0.554,
                                "type_hierarchy": "/animal/invertebrate/shellfish  /conch"
                            },
                            {
                                "class": "jade green color",
                                "score": 0.939
                            },
                            {
                                "class": "greenishness color",
                                "score": 0.856
                            }
                        ],
                        "classifier_id": "default",
                        "name": "default"
                    }
                ],
                "image": "1011328.jpg"
            }
        ],
        "images_processed": 1
    }

Since it is not satisfying for food classification, we'll have to create our own classifier.

# Create a custom classifier

Before uploading images to the classifier, we need to verify that each zip file is no more than 100 MB and 10 000 images per .zip file, no less than 10 images per size.

```bash
du -h food-101/images/*.zip
```

Each zip file is around 25MB (well balanced dataset - have the same number of images per class). Each zip file should not be above 100MB, otherwise we would have to divide them into multiple zip files.

Create the classifier under the name **food-101**. For creation, the service accepts a maximum of 300 MB so we can only upload up to 10 classes per update.

```bash
DATA=`pwd`/food-101/images
FIRST_CLASSES=`ls train/*.zip | awk 'NR >=1 && NR <=10 {split($0,a,"."); split(a[1],a,"/"); printf " -F " a[2] "_positive_examples=@" $0 }' -  `
echo $FIRST_CLASSES
```
returns what we need :
    -F apple_pie_positive_examples=@food-101/images/apple_pie.zip -F baby_back_ribs_positive_examples=@food-101/images/baby_back_ribs.zip -F baklava_positive_examples=@food-101/images/baklava.zip -F beef_carpaccio_positive_examples=@food-101/images/beef_carpaccio.zip -F beef_tartare_positive_examples=@food-101/images/beef_tartare.zip -F beet_salad_positive_examples=@food-101/images/beet_salad.zip -F beignets_positive_examples=@food-101/images/beignets.zip -F bibimbap_positive_examples=@food-101/images/bibimbap.zip -F bread_pudding_positive_examples=@food-101/images/bread_pudding.zip -F breakfast_burrito_positive_examples=@food-101/images/breakfast_burrito.zip

Let's upload them :

```bash
curl -X POST $FIRST_CLASSES -F "name=food-101" "https://gateway-a.watsonplatform.net/visual-recognition/api/v3/classifiers?api_key=$API_KEY&version=2016-05-20" > response.json
```

You can list your existing classifiers :

```bash
curl -X GET "https://gateway-a.watsonplatform.net/visual-recognition/api/v3/classifiers?api_key=$API_KEY&version=2016-05-20"
```

which gives

    {"classifiers": [
    {
        "classifier_id": "food101_1404391194",
        "name": "food-101",
        "status": "training"
    }
    ]}

or parse `response.json` to get the classifier ID :

    {
    "classifier_id": "food101_1404391194",
    "name": "food-101",
    "owner": "de67af8c-862c-4002-88a8-259653c880c4",
    "status": "training",
    "created": "2016-12-21T15:32:10.458Z",
    "classes": [
        {"class": "beef_carpaccio"},
        {"class": "baklava"},
        {"class": "baby_back_ribs"},
        {"class": "apple_pie"},
        {"class": "bibimbap"},
        {"class": "beignets"},
        {"class": "beet_salad"},
        {"class": "beef_tartare"},
        {"class": "breakfast_burrito"},
        {"class": "bread_pudding"}
    ]
    }


```bash
CLASSIFIER=`cat response.json | python -c "import json,sys;obj=json.load(sys.stdin);print obj['classifier_id'];"`
```

You can retrieve more info about the classifier :

```bash
curl -X GET "https://gateway-a.watsonplatform.net/visual-recognition/api/v3/classifiers/$CLASSIFIER?api_key=$API_KEY&version=2016-05-20"
```

    {
        "classifier_id": "food101_1404391194",
        "name": "food-101",
        "owner": "de67af8c-862c-4002-88a8-259653c880c4",
        "status": "ready",
        "created": "2016-12-21T15:32:10.458Z",
        "classes": [
            {"class": "beef_carpaccio"},
            {"class": "baklava"},
            {"class": "baby_back_ribs"},
            {"class": "apple_pie"},
            {"class": "bibimbap"},
            {"class": "beignets"},
            {"class": "beet_salad"},
            {"class": "beef_tartare"},
            {"class": "breakfast_burrito"},
            {"class": "bread_pudding"}
        ]
    }


# Update the created classifier with more classes

Let's upload the remaining 91 classes. The service accepts a maximum of 256 MB per training call so we can only upload up to 9 classes per update. Also, if you submit an update command without waiting the **ready** state, the previous update command will be erased/canceled, which is not an intented behavior. After the first update, a retrained timestamp appears in the classifier JSON. After each retraining, the retrained timestamp will be updated with the last retraining update :

```bash
for i in {0..10}; do

  while [ "$STATUS" != "ready" ]
  do
    STATUS=`curl -s -X GET "https://gateway-a.watsonplatform.net/visual-recognition/api/v3/classifiers/$CLASSIFIER?api_key=$API_KEY&version=2016-05-20" | python -c "import json,sys;obj=json.load(sys.stdin);print obj['status'];"`
    echo "Not ready. Waiting 10s."
    sleep 10s
  done


  CLASS_LIST=`ls train/*.zip | awk -v first="$(($i*9+10 + 1))" -v last="$(($i*9+10 + 9))" 'NR >=first && NR <=last {split($0,a,"."); split(a[1],a,"/"); printf " -F " a[2] "_positive_examples=@" $0 }' -  `

  COMMAND="curl -X POST $CLASS_LIST https://gateway-a.watsonplatform.net/visual-recognition/api/v3/classifiers/$CLASSIFIER?api_key=$API_KEY&version=2016-05-20"
  echo $COMMAND
  $COMMAND
done
```

# Classify an image with the custom classifier

    curl -X POST -F "images_file=@test/apple_pie/1011328.jpg" "https://gateway-a.watsonplatform.net/visual-recognition/api/v3/classify?api_key=$API_KEY&classifier_ids=$CLASSIFIER&version=2016-05-20"

It is possible to request multiple classifiers at the same time, by separating classifier IDs with a comma. The default classifier is 'Default'.


# Delete the custom classifier

Delete classifier :

```bash
curl -X DELETE "https://gateway-a.watsonplatform.net/visual-recognition/api/v3/classifiers/$CLASSIFIER?api_key=$API_KEY&version=2016-05-20"

```

**Well done!**