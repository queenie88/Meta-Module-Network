# Meta-Module-Network
Code for WACV 2021 Paper "Meta Module Network for Compositional Visual Reasoning"

<p>
<img src="architecture.png" width="800">
</p>

This repository contains the following components:
1. The generated programs from the program generator.
2. The symbolic execution for the visual question answering.
3. For evaluation code, please refer to the code/ folder. You need to download the code from google drive: https://drive.google.com/open?id=14sKn_163FYqTVInV6OL_ZukT2wp2uPtn, the instruction to run the code is provided under the code/ folder.

## Function Definition
We define roughly 20+ functions based on the semantic-str provided in the original GQA dataset and categorize them into the following classes:
1. relate: finding objects with name and relation constraint, two version: normal + reverse.
2. filter: filter objects with given attributes or filter objects based on horizontal or vertical geometric position. 
3. filter_not: filter objects without given attributes
4. query: query object names or attributes or positions in the scene graph
5. verify: verify whether the objects contain certain attributes or their horizontal/vertial positions.
6. verify_rel: verify relations between different objects in the image.
7. choose: choose which attributes or names or geometric locations the current object has.
8. choose_rel: choose which relation the objects have.
9. and/or/exist: logical operations.
10: different: whether the objects are different
11: same: whether the objects are the same.
12: commmon: what attributes the objects have in common.
13: same_attr: whether the objects have the same given attributes.
14: different_attr: whether the objects have different given attributes.

## Description of different files
- API_provider.py: define the functions inside the API
- sceneGraphs/trainval_bounding_box.json: the scene graph provided by the original GQA dataset
  ```
    {
      imageId:
      {
        bouding_box_id:
        {
          x: number,
          y: number,
          w: number,
          h: number,
          relations: [{object: "bounding_box_id", name: "relation_name"} ... ],
          name: object_class,
          attributes: [attr1, attr2, ... ]
        },
        bouding_box_id:
        {
          ...
        },
      }
    }
  ```
- GQA_hypernym.py: define the hypernym/hyponym we used in string matching during execution
- questions: the questions-program pairs and their associated images.
  ```
  [
    [
      "ImageId",
      "Question",
      "Programs": [f1, f2, ..., fn],
      "QuestionId",
      "Answer"
    ]
  ]
  ```

## Data Downloading
Download all the question files and scene graph files ready for symbolic execution using
  ```
    bash get_data.sh
  ```
This script will download questions/ folder, and the "trainval_all_programs.json" is used for bootstrapping and "trainval_unbiased_programs.json" is used for finetunning in the paper. The "trainval_unbiased_programs.json" and "testdev_pred_programs.json" are both generated by the program generator model, which contains some errors.

## Symbolic Execution
We can run the run.py to perform symbolic execution on the GQA provided scene graph to get the answer.
  ```
    python run.py --do_trainval_unbiased
  ```
The script will return 
  ```
  success rate (ALL) = 0.923610917323838, success rate (VALID) = 0.9605288360151759, valid/invalid = 1033742/41320
  ```
It means that for those questions, whose answer is inside the scene graph, the accuracy is 96%. There are 4% of questions without answers inside the scene graph, therefore the overall accuracy is 92.3%. This is good enough as a symbolic teacher to teach the meta module network to reason.


# Meta Module Network
All the training data (under the questions/ folder) given to the networks are called '*_inputs.json', these files are simply a restructured data format (containing the dependency between the execution from different steps) from the original "*_programs.json" files. For example, the 'trainval_balanced_programs.json' can be used to generate 'trainval_balanced_inputs.json' using the script in https://github.com/wenhuchen/Meta-Module-Network/blob/master/code/preprocess.py.

- *_programs.json
```
    "2354786",
    "Is the sky dark?",
    [
      "[2486325]=select(sky)",
      "?=verify([0], dark)"
    ],
    "02930152",
    "yes"
```
- *_inputs.json
```
    "2354786",
    "Is the sky dark?",
    [],
    [
      [
        "select",
        null,
        null,
        null,
        "sky",
        null,
        null,
        null
      ],
      [
        "verify",
        null,
        "dark",
        null,
        null,
        null,
        null,
        null
      ]
    ],
    [
      [], 
      [
        [
          1,
          0
        ]
      ]
    ],
    "02930152",
    "yes"
```

In the input file, the following data type is called program recipe, corresponding to "[2486325]=select(sky)".
```
[
  "select",
  null,
  null,
  null,
  "sky",
  null,
  null,
  null
],
```
In the input file, the following data type is called layer dependency. In the 0-th element (1st layer of MMN), there is only a [], which means nothing is dependent on the previous layer. In the 1-th element (2nd layer of MMN), there is a [1, 0], which means that the 1-st node's is dependent on 0-th node's output (e.g. "?=verify([0], dark)") in this layer. 
```
    [
      [], 
      [
        [
          1,
          0
        ]
      ]
    ],
```
The dependency relationship is the critical part in MMN, for example, the dependency of "[[], [[1,0], [3,0]], [[2,1]], [[4,2], [4,3]]]" is visualized as below: 
<p>
<img src="introduction.png" width="800">
</p>
