# Scoring in Direct Runs

This section contains a few extra details about the Direct Runs supported by the FireDrone API.

The first step to use the API is installing the FireDrone API SDK (for example, from an Azure Notebook):

```python
!pip install --index-url https://test.pypi.org/simple/ fire-drone-sdk -U
```

First, you need to get hold of your `Workspace` and check the list of active scenes:

```python
import firedrone.client as fdc
from firedrone.client.errors import FireDroneClientHttpError
import os

api_key = '...'  #replace with your own API key

workspace = fdc.Workspace(api_key)

scenes = workspace.get_scenes()
print(scenes)
```

The following table displays the secenes that are active as of July 31, 2019.

Scene Id | Scene Name | Description
--- | --- | ---
20 | No-fire scene | Initial scene that does not contain any real fire.
21 | Notre-Dame in flames | A scene from the fire that engulfed Note-Dame in Paris.
22 | Misleading Scene | This is a beautiful sunset (also real). Unfortunatelly, it looks a lot like fire in the sky.
23 | Large Street Fire Scene | A large street fire in a city. The fire has a complex structure which blends with some smoke patterns.
24 | City Fire with Smoke Scene | An area of a city engulfed in flames. One large fire and several smaller ones can be spotted.

**Note**

We will publish a special scene that you must use to submit your official scorings for the contest. Watch email and the dedicated Slack channel for the announcement regarding the publication of this particula scene.

## Scoring a scene

The details about starting a Direct Run and performing different actions (e.g. moving the virtual drone) are presented in detail in the `DirectRuns.ipynb` notebook. This section focuses on providing extra details about the scoring process itself.

The scoring is always performed against the current field of view of the drone. Remember, you can get that view at any point in time using (example from an Azure Notebook):

```python
# Get the current drone field of view image
# -------------------------------------------------------------------------------------------

frame = workspace.get_drone_fov_image(run_id)
with open('./frame.png', 'wb') as f:
   f.write(frame)

from IPython.display import Image
Image(filename='frame.png') 
```

There are two different types of scoring you can perform:
- True/False (simple) scoring - you classify the image you get from the drone as a Fire/No-Fire scene.
- Pixel (complex) scoring - you classify each pixel from the image you get from the drone as a Fire/No-Fire pixel.

Here is an example containing both types of scoring:

```python
# Score the drone field of view image
# -------------------------------------------------------------------------------------------
score_result = workspace.directrun_score(run_id, True)
print(score_result)

# Score the drone field of view image pixels
# -------------------------------------------------------------------------------------------
pixels_on_fire = [1,0,0,1,0,1,1,1,1,1,1,1,1,1,1,1]
score_result = workspace.directrun_score_pixels(run_id, pixels_on_fire)
print(score_result)
```

In the case of pixel scoring, we will expect by default to get a list of ones and zeros that has the length equal to the total number of pixels in the image. Numbering starts from the top left corner of the image and covers each line of pixels until the bottom right corner of the image is reached. In addition, the following rules apply:

- Any value other then 0 or 1 is considered an outlier and is automatically replaced with 0 (no-fire).
- If the length of the list is larger than the total number of pixels, it will be automatically truncated to the right size. For example, given an image of 200 x 200 pixels, any element in the list past the 40,000th position will be ignored.
- If the length of the list if smaller than the total number of pixels, we will automatically pad it with 0 (no-fire). For example, given an image of 200 x 200 pixels and a scoring list of 32,000 elements, the list will be automatically extended with an extra 8,000 elements (all zeros).

## Scoring in the context of a Direct Run

Each time you start a Direct Run, the drone position is reset to its default which is the horizontaly centered at the bottom of the scene.

Subsequent movements of the drone will move it up, down, left, or right. Any attempt to move outside of the scene will end up in getting a `{'success': False}` result indicating that the move was not executed.

For each position you reach, you can decide to:
- Not score at all. This typically happens when your model tells you there is no fire in the image.
- Performe simple scoring. This typically happens when your model tells you there is fire in the image.
- Perform complex scoring. This typicall happens when your model tells you there is fire in the image and it also has the capability of identifying the extent of that fire.
- Perform both simple and complex scoring.

For each position you don't reach, it is implicitly assumed that there is no scoring.

For a given position of the drone, the following actions are equivalent:
- Not scoring at all
- Using simple scoring to classify the image as No-Fire
- Using complex scoring to classify all pixels as No-Fire (all zeros)

In other words, if your model tells you there is no fire, you don't need to score at all.

A complex scoring that has at least one value of 1 in the list also qualifies as a simple scoring with outcome Fire.

## Evaluating your scoring

The following rules apply:

- The only scene used for official evaluation is the one we will publish at the end of the contest and point out to be as the `evaluation scene`. The rest of your Direct Runs are only for testing purposes and they will not be taken into consideration. They are there to help you get familiar with the platform and prepare for that so-important final run.
- There has to be a demonstrable correlation between the code you submit as a solution and the results recorded during the Direct Run on the scene used for final evaluation.
- Only one Direct Run is accepted as an official submission on the `evaluation scene` and used for final evaluation. In case there are multiple Direct Runs on the scene, only the first one will be taken into consideration. In case there is a technical problem you encounter during the Direct Run, you will need to contact us directly to adress the issue.
- First, we will evaluate the Fire/No-Fire classification (a.k.a. simple scoring). The following drone positions will be taken into consideration:
  - Positions on which you submitted a simple scoring with outcome Fire
  - Positions on which you didn't submit a simple scoring but you submitted a complex scoring that has at least one value of 1 (Fire) in the list.
  


  We will count the following:
  - True Positives (TPs) - drone positions for which you correctly identified a fire in the drone's field of view images.
  - False Positives (FPs) - drone positions for which you incorrectly identified a fire in the drone's fied of view images.

  Our evaluation will take into account the following:
  - The more TPs you have, the higher the rank
  - The less FPs you have, the higher the rank

  We fully undestand that the two rules above might get into conflict and this is where our judgement and own experience will kick in. The ideal case is when your TPs equal the actual number of positions where fire is detected and you have zero FPs (otherwise, you could get the necessary TPs by simply scoring all positions with Fire). A good case is when you have a high number of TPs and a small number of FPs. Suppose we have 6 position where fire is actually present, let's look at some potential examples of evaluation:

  - 6 TPs + 0 FPs - perfect score, ideal :)
  - 5 TPs + 2 FPs - good score, excellent
  - 5 TPs + 2 FPs wins over 6TPs + 10 FPs but not over 6TPs + 2 FPs.
  - 5 TPs + 2 FPs wins over 5TPs + 3 or more FPs
- In cases where it is difficult to differentiate based on simple scoring only, we will evaluate the Pixel Fire/No-Fire classification. The same principle applies, the more TPs and less FPs you have, the better your rank. This means that you are highly encouraged to do pixel scoring as well as it might help you win in cases where simple scoring makes you equal with other participants.

## A final note on evaluation

As mentioned above, we might get into situations where exact mathematical rules and figures will not be able to properly differentiate two or more solutions. In these cases we will have to resort to our own judgement and experience.

Most importantly, the evaluation of the scoring in Direct Runs is only a part of the overal scoring. We encourage you to check the **JUDGING CRITERIA** from the contest's official [DevPost page](https://firedrone.devpost.com/) to learn about the broader context in which your solution will be evaluated.