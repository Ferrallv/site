---
title: "BigBrainBeat - A neural network to detect BPM"
excerpt_separator: "<!--more-->"
categories:
  - Project
---



___

## *TL;DR (or <u>Don't talk tech to me</u>)*

I set out with the goal to find the BPM of a music sample using an AI (specifically a Convolution Neural Network). It turned out to be pretty successful with the data I used.

___

## Overview

It will be my hope to explain: what my goal was, my trials and tribulations in the process as well as my reasoning for some of the choices made, and how to recreate the experiment, results, and presentation. It may be nice to follow along with the [code](https://github.com/Ferrallv/BigBrainBeat).

## Goal

I set out to build an AI that could determine the beats within a piece of music. In this context I loosely defined a "beat" as the time in music when you nod your head, snap your fingers, tap your foot, etc. If unfamiliar with related musical theory, an easy explanation can be found [here](https://www.bbc.co.uk/bitesize/guides/zs9wk2p/revision/1).

I got the idea for the method from [Liam Schoneveld](https://github.com/nlml/bpm)'s work, which I found when researching this idea.

## Versions 1 and 2

I started initially with a lot of failures. While developing my Convolution Neural Network I approached different methods of formatting my data.

In version 1 I was attempting to use a [Short-time Fourier transform](https://en.wikipedia.org/wiki/Short-time_Fourier_transform) of a `.wav` file to create an image, which would then be used to predict a vector of equal time steps to the time steps of the audio.

To explain, a wave file generally has 44100 records per second of audio (the sample rate). To emulate the location of a beat, a vector was created to match the audio file, where 1 was entered to indicate that a beat should be at this time, and a 0 otherwise.

To demonstrate, a target vector for a 4 second slice of music, played at 60 BPM (one beat per second), at a sample rate of 4, and starting on the beat would look like:

<p align="center">[1,0,0,0,1,0,0,0,1,0,0,0,1,0,0,0]</p>

Using target vectors like this at higher sample rates proved to be difficult for my models to predict and I had little success. I went back to the drawing board.

I thought about the way we hear the beat in music. A beat is an event in time that we recognize in relation to the changes that happen temporally in music. A frequency change or an amplitude change <u>in time</u> is more important to determining the rhythm than the degree to which they change.<sup>[1](#fn1)</sup>
 So I began to simplify my data to highlight these changes.

Version 2 experimented with 1D convolution filters, explained in the [keras documentation](https://keras.io/layers/convolutional/), to be commonly used to highlight temporal changes. Here I experimented with different layers, such as [BatchNormalization](https://keras.io/layers/normalization/), [MaxPooling](https://keras.io/layers/pooling/), and multiple [DenseLayers](https://keras.io/layers/core/). These trials lead me to my most successful model.

## Version 3

#### The Data

When considering the creation of the neural network I had to decide what data I would use. I hypothesized that a model might find more success if I train it on simple data first and then, using transfer learning, progressed onto more complicated data. I have not yet tested to see if this method would be any more or less effective than just starting on more complicated data.

In Phase 1 I wrote 80 drum tracks, with BPM ranging from 80-160.

- I chose this BPM range because:
    -  My theory is that the qualities that distinguish a beat are independent of the distance between another beat.

    -  Music that has either half or double the BPM of other music that started at the same time will have their beats match up every time for the slower piece, and every second beat for the faster. To illustrate:

      
      60 BPM, sample rate = 4

      [1,0,0,0,1,0,0,0,1,0,0,0,1,0,0,0]

      120 BPM, sample rate = 4

      [1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0]

      

With this reasoning in mind, I felt that I effectively covered BPM between 40-320.

The phase 1 `.wav` files were simply a drum kick sound for every beat. The created tracks were ~10 seconds in length. By creating the music myself I was able to insure that the beats were exactly in time with the prescribed BPM and consistently so.

In Phase 2 I created a cacophony of drums and synths. It was also created digitally to be consistent and precisely in time. The rhythms and sounds changed erratically throughout. Every drum sound was synced to a segmentation of 1/16<sup>th</sup> of a beat. The BPM range in phase 2 is the same as that in phase 1. The tracks length are similarly ~10 seconds.

In Phase 3 I used the music listed in the `SourcesAndInspirations.md` found [here](https://github.com/Ferrallv/BigBrainBeat/blob/master/SourcesAndInspirations.md). For those interested, I converted each file into `.wav` using [this](https://librosa.github.io/librosa/generated/librosa.output.write_wav.html). I could not be positive what the exact BPM was for each song, instead tapping the tempo myself 50 times and taking the average speed using [this website](https://www.all8.com/tools/bpm.htm).

#### The Model & Process

I needed to manipulate the raw `.wav` files into suitable shapes for the CNN, as well as to create enough data to use for training and testing the model. To do this I wrote the function `makedf`.

`makedf` extracts the data from a `.wav` file as a numpy array, slices it up randomly a given number of times, and returns these slices as matrices (for the CNN).


{% raw %}<img src="/assets/images/downsampled90bpmwav.png" alt="">{% endraw %}

A visualization of a slice from an 90 BPM phase 1 `.wav`


I wrote a `for` loop to go through all of the tracks that I made for that specific phase to create the dataset. The data was then split into train and test sets.

You will notice that each `.wav` is normalized between 0 and 1 in the `makedf` function. Further on,  the train and test data are "normalized" again. I did this to emphasize the maximum values within the the file in relation to the rest of the data. I theorized that this would help to capture the "downbeat", a characteristic moment that often captures the beginning of a musical phrase.

The model itself consisted of two `1D convolution` layers with ten filters and kernal sizes of 3 and 5, respectively. Then there was a flattening before two `Dense` layers of equal length (80 classes, for each BPM from 80 to 160). The activation for every layer is `relu` until the last which was `softmax`. There was a `Dropout` layer of 0.5 after each layer except `Flatten` and the final `Dense`. I found this set up to be the most successful throughout experimentation.

Our problem is multi-class classification (determining the BPM) and so I used the loss function `sparse_categorical_crossentropy` as the function whose minimization would lead us to a better model. The optimizer chosen was `Adam` as it was what performed best during earlier experimentation. `Accuracy` was the goal metric of this model.

After creating an EC2 instance with a [DLAMI](https://docs.aws.amazon.com/dlami/latest/devguide/what-is-dlami.html) on AWS I was able to access an environment that was set up for [multi-threading](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-optimize-cpu.html) so that I was able to use all of the vCPUs available for training my model. When I fit my model I chose a batch size of 50, simply because that evenly divided my training set. Because of the computing power I now had, I chose to train my model for 50 epochs. I implemented an `EarlyStopping` clause to cease after no reduction in the validation loss after two epochs.

In each phase, the model and its weights from the previous phase were imported and used as a starting point for the new phase of training.

#### Results

In phase 1 we see an accuracy of 63%, phase 2: 59%, and phase 3: 90%. The accuracy in the third model being higher is likely explained by the lack of variety of tempo in the music that was used.

The mapped accuracy and loss scores over the epochs indicate that the model is successful with little over-fitting.

{% raw %}<img src="/assets/images/BigBrainBeatv3_phase1_accAndLoss.png" alt="">{% endraw %}
{% raw %}<img src="/assets/images/BigBrainBeatv3_phase2_accAndLoss.png" alt="">{% endraw %}
{% raw %}<img src="/assets/images/BigBrainBeatv3_phase3_accAndLoss.png" alt="">{% endraw %}


## Conclusions

Possible applications of the model might be:

   - As another feature in identity security (assuming that each person has a unique beat pattern to their speech).
   - As a feature in determining emotion or mental state (in relation, if someones speech speeds up it may indicate anxiety, while maybe slower speech could indicate inebriation).
   - Media applications when matching different events in time with music.
   - Nerding out with friends as we yell into my computer microphone to see what the BPM of our jibber-jabber might be.

I would like to view this model as a proof of concept for using a CNN to determine the BPM of music.

To be frank, there are likely methods that are simpler and more accurate. This might be a case of using a sledgehammer to kill a mosquito.

In the future I hope to explore different ways of determining the beat in music with fluctuating BPM and inconsistencies of beat locations in time. The next step would be to find a reasonable method to find where the beat is in time instead of inferring it from the BPM and an "onset" (the sound peak or max that occurs when the sound instantiates) as demonstrated in the presentation notebook in my github repo.

## Presentation

To see the models in action/application, run the `BigBrainBeat_Presentation.ipynb` found at the [BigBrainBeat](www.github.com/ferrallv/bigbrainbeat) repo. There you will also find a link to the `.wav` files I used. I encourage you to try your own `.wav` files as well to determine the success of the model yourself!

---

<a name="fn1">1</a>: Bouwer FL, Van Zuijen TL, Honing H (2014) Beat Processing Is Pre-Attentive for Metrically Simple Rhythms with Clear Accents: An ERP Study. PLoS ONE 9(5): e97467. https://doi.org/10.1371/journal.pone.0097467
