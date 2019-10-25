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

    [1,0,0,0,1,0,0,0,1,0,0,0,1,0,0,0]

Using target vectors like this at higher sample rates proved to be difficult for my models to predict and I had little success. I went back to the drawing board.

I thought about the way we hear the beat in music. A beat is an event in time that we recognize in relation to the changes that happen temporally in music. A frequency change or an amplitude change <u>in time</u> is more important to determining the rhythm than the degree to which they change.<sup>[1](#fn1)</sup>
 So I began to simplify my data to highlight these changes.

Version 2 experimented with 1D convolution filters, which are explained in the [keras documentation](https://keras.io/layers/convolutional/) to be commonly used with temporal inputs. Here I experimented with different layers, such as [BatchNormalization](https://keras.io/layers/normalization/), [MaxPooling](https://keras.io/layers/pooling/), and multiple [DenseLayers](https://keras.io/layers/core/). These trials lead me to my most successful model.

## Version 3

### The Data

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

### The Model & Process

I needed to manipulate the raw `.wav` files into suitable shapes for the CNN, as well as to create enough data to use for training and testing the model. To do this I wrote the function `makedf`.

`makedf` extracts the data from a `.wav` file as a numpy array, slices it up randomly a given number of times, and returns these slices as matrices (for the CNN).

![A visualization of a slice from an 90 BPM phase 1 `.wav`]({{site.url}}{{site.baseurl}}/assets/images/downsampled90bpmwav.png)

A visualization of a slice from an 90 BPM phase 1 `.wav`


I wrote a `for` loop to go through all of the tracks that I made for that specific phase to create the dataset. The data was then split into train and test sets.

You will notice that each `.wav` is normalized between 0 and 1 in the `makedf` function. Further on,  the train and test data are "normalized" again. I did this to emphasize the maximum values within the the file in relation to the rest of the data. I theorized that this would help to capture the "downbeat", a characteristic moment that often captures the beginning of a musical phrase.

The model itself consisted of two `1D convolution` layers with ten filters and kernal sizes of 3 and 5, respectively. Then there was a flattening before two `Dense` layers of equal length (80 classes, for each BPM from 80 to 160). The activation for every layer is `relu` until the last which was `softmax`. There was a `Dropout` layer of 0.5 after each layer except `Flatten` and the final `Dense`. I found this set up to be the most successful throughout experimentation.

Our problem is multi-class classification (determining the BPM) and so I used the loss function `sparse_categorical_crossentropy` as the function whose minimization would lead us to a better model. The optimizer chosen was `Adam` as it was what performed best during earlier experimentation. `Accuracy` was the goal metric of this model.

After creating an EC2 instance with a [DLAMI](https://docs.aws.amazon.com/dlami/latest/devguide/what-is-dlami.html) on AWS I was able to access an environment that was set up for [multi-threading](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-optimize-cpu.html) so that I was able to use all of the vCPUs available for training my model. When I fit my model I chose a batch size of 50, simply because that evenly divided my training set. Because of the computing power I now had, I chose to train my model for 50 epochs. I implemented an `EarlyStopping` clause to cease after no reduction in the validation loss after two epochs.

In each phase, the model and its weights from the previous phase were imported and used as a starting point for the new phase of training.

### Results

In phase 1 we see an accuracy of 63%, phase 2: 59%, and phase 3: 90%. The accuracy in the third model being higher is likely explained by the lack of variety of tempo in the music that was used.

The mapped accuracy and loss scores over the epochs indicate that the model is successful with little over-fitting.

<html lang="en">

  <head>

      <meta charset="utf-8">
      <title>The accuracy and loss over epochs in phase 1</title>







        <script type="text/javascript" src="https://cdn.pydata.org/bokeh/release/bokeh-1.3.4.min.js"></script>
        <script type="text/javascript">
            Bokeh.set_log_level("info");
        </script>




  </head>


  <body>






              <div class="bk-root" id="c0277cb6-8bc7-4251-84c5-84345b6cca14" data-root-id="16824"></div>





        <script type="application/json" id="17436">
          {"9dc03127-455d-40c1-85b1-19d65f6ebbe3":{"roots":{"references":[{"attributes":{"children":[{"id":"16752","subtype":"Figure","type":"Plot"},{"id":"16788","subtype":"Figure","type":"Plot"}]},"id":"16824","type":"Row"},{"attributes":{"line_color":"blue","line_width":2,"x":{"field":"epoch"},"y":{"field":"loss"}},"id":"16811","type":"Line"},{"attributes":{"index":1,"label":{"value":"Validation Accuracy"},"renderers":[{"id":"16782","type":"GlyphRenderer"}]},"id":"16785","type":"LegendItem"},{"attributes":{},"id":"17221","type":"UnionRenderers"},{"attributes":{"ticker":{"id":"16762","type":"BasicTicker"}},"id":"16765","type":"Grid"},{"attributes":{"data_source":{"id":"16750","type":"ColumnDataSource"},"glyph":{"id":"16780","type":"Line"},"hover_glyph":null,"muted_glyph":null,"nonselection_glyph":{"id":"16781","type":"Line"},"selection_glyph":null,"view":{"id":"16783","type":"CDSView"}},"id":"16782","type":"GlyphRenderer"},{"attributes":{"axis_label":"Epoch","formatter":{"id":"17214","type":"BasicTickFormatter"},"ticker":{"id":"16762","type":"BasicTicker"}},"id":"16761","type":"LinearAxis"},{"attributes":{"line_alpha":0.1,"line_color":"#1f77b4","line_width":2,"x":{"field":"epoch"},"y":{"field":"val_acc"}},"id":"16781","type":"Line"},{"attributes":{},"id":"16767","type":"BasicTicker"},{"attributes":{"callback":null},"id":"16753","type":"DataRange1d"},{"attributes":{"data_source":{"id":"16750","type":"ColumnDataSource"},"glyph":{"id":"16811","type":"Line"},"hover_glyph":null,"muted_glyph":null,"nonselection_glyph":{"id":"16812","type":"Line"},"selection_glyph":null,"view":{"id":"16814","type":"CDSView"}},"id":"16813","type":"GlyphRenderer"},{"attributes":{},"id":"16757","type":"LinearScale"},{"attributes":{"source":{"id":"16750","type":"ColumnDataSource"}},"id":"16814","type":"CDSView"},{"attributes":{"text":"Loss by Epoch for BigBrainBeatv3_phase1"},"id":"16809","type":"Title"},{"attributes":{"index":0,"label":{"value":"Training Accuracy"},"renderers":[{"id":"16813","type":"GlyphRenderer"}]},"id":"16820","type":"LegendItem"},{"attributes":{},"id":"16762","type":"BasicTicker"},{"attributes":{"text":"Accuracy by Epoch for BigBrainBeatv3_phase1"},"id":"16773","type":"Title"},{"attributes":{"axis_label":"Accuracy","formatter":{"id":"17212","type":"BasicTickFormatter"},"ticker":{"id":"16767","type":"BasicTicker"}},"id":"16766","type":"LinearAxis"},{"attributes":{"line_color":"orange","line_width":2,"x":{"field":"epoch"},"y":{"field":"val_loss"}},"id":"16816","type":"Line"},{"attributes":{"dimension":1,"ticker":{"id":"16767","type":"BasicTicker"}},"id":"16770","type":"Grid"},{"attributes":{"line_alpha":0.1,"line_color":"#1f77b4","line_width":2,"x":{"field":"epoch"},"y":{"field":"val_loss"}},"id":"16817","type":"Line"},{"attributes":{"below":[{"id":"16797","type":"LinearAxis"}],"center":[{"id":"16801","type":"Grid"},{"id":"16806","type":"Grid"},{"id":"16822","type":"Legend"}],"left":[{"id":"16802","type":"LinearAxis"}],"plot_height":400,"renderers":[{"id":"16813","type":"GlyphRenderer"},{"id":"16818","type":"GlyphRenderer"}],"title":{"id":"16809","type":"Title"},"toolbar":{"id":"16807","type":"Toolbar"},"x_range":{"id":"16789","type":"DataRange1d"},"x_scale":{"id":"16793","type":"LinearScale"},"y_range":{"id":"16791","type":"DataRange1d"},"y_scale":{"id":"16795","type":"LinearScale"}},"id":"16788","subtype":"Figure","type":"Plot"},{"attributes":{"line_color":"blue","line_width":2,"x":{"field":"epoch"},"y":{"field":"acc"}},"id":"16775","type":"Line"},{"attributes":{"data_source":{"id":"16750","type":"ColumnDataSource"},"glyph":{"id":"16816","type":"Line"},"hover_glyph":null,"muted_glyph":null,"nonselection_glyph":{"id":"16817","type":"Line"},"selection_glyph":null,"view":{"id":"16819","type":"CDSView"}},"id":"16818","type":"GlyphRenderer"},{"attributes":{"source":{"id":"16750","type":"ColumnDataSource"}},"id":"16778","type":"CDSView"},{"attributes":{},"id":"16795","type":"LinearScale"},{"attributes":{},"id":"17218","type":"BasicTickFormatter"},{"attributes":{"source":{"id":"16750","type":"ColumnDataSource"}},"id":"16819","type":"CDSView"},{"attributes":{},"id":"16759","type":"LinearScale"},{"attributes":{},"id":"16793","type":"LinearScale"},{"attributes":{"callback":null},"id":"16755","type":"DataRange1d"},{"attributes":{"items":[{"id":"16784","type":"LegendItem"},{"id":"16785","type":"LegendItem"}],"location":"bottom_right"},"id":"16786","type":"Legend"},{"attributes":{"callback":null,"tooltips":[["Validation Accuracy","@val_acc"],["Accuracy","@acc"],["Epoch","@epoch"]]},"id":"16751","type":"HoverTool"},{"attributes":{"line_alpha":0.1,"line_color":"#1f77b4","line_width":2,"x":{"field":"epoch"},"y":{"field":"loss"}},"id":"16812","type":"Line"},{"attributes":{"index":0,"label":{"value":"Training Accuracy"},"renderers":[{"id":"16777","type":"GlyphRenderer"}]},"id":"16784","type":"LegendItem"},{"attributes":{"line_alpha":0.1,"line_color":"#1f77b4","line_width":2,"x":{"field":"epoch"},"y":{"field":"acc"}},"id":"16776","type":"Line"},{"attributes":{"index":1,"label":{"value":"Validation Accuracy"},"renderers":[{"id":"16818","type":"GlyphRenderer"}]},"id":"16821","type":"LegendItem"},{"attributes":{"callback":null},"id":"16789","type":"DataRange1d"},{"attributes":{"items":[{"id":"16820","type":"LegendItem"},{"id":"16821","type":"LegendItem"}]},"id":"16822","type":"Legend"},{"attributes":{"below":[{"id":"16761","type":"LinearAxis"}],"center":[{"id":"16765","type":"Grid"},{"id":"16770","type":"Grid"},{"id":"16786","type":"Legend"}],"left":[{"id":"16766","type":"LinearAxis"}],"plot_height":400,"renderers":[{"id":"16777","type":"GlyphRenderer"},{"id":"16782","type":"GlyphRenderer"}],"title":{"id":"16773","type":"Title"},"toolbar":{"id":"16771","type":"Toolbar"},"x_range":{"id":"16753","type":"DataRange1d"},"x_scale":{"id":"16757","type":"LinearScale"},"y_range":{"id":"16755","type":"DataRange1d"},"y_scale":{"id":"16759","type":"LinearScale"}},"id":"16752","subtype":"Figure","type":"Plot"},{"attributes":{"callback":null},"id":"16791","type":"DataRange1d"},{"attributes":{},"id":"17212","type":"BasicTickFormatter"},{"attributes":{"ticker":{"id":"16798","type":"BasicTicker"}},"id":"16801","type":"Grid"},{"attributes":{"callback":null,"data":{"acc":[0.01234567873639825,0.04567901160573109,0.08998628196770271,0.12098765384983297,0.14828532217747553,0.17462277051127822,0.19574760031389438,0.2021947877969598,0.2198902607393363,0.23113854624974875,0.2447187927626942,0.24814814758398895,0.2674897110552127,0.27572016635672053,0.2733882029422026,0.29039780607521126,0.2814814832025758,0.29259259246586117,0.29986282674128134,0.3080932789562661,0.3075445819497926,0.31385459713075714,0.32441701094105085,0.32729766918512365,0.3251028816089218],"epoch":[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25],"loss":[4.405586145869335,4.128188584567098,3.5218306454447887,3.1002906726057473,2.794490026675461,2.5563043841609248,2.4130283304693276,2.2969682857659945,2.1995710638161388,2.134969172163755,2.0921165326658753,2.0442344048386247,1.991390266699719,1.934269661438972,1.9422965761074804,1.8854667896760018,1.882469892501831,1.843233526488881,1.817847029335378,1.8060229895700317,1.782602929120528,1.773487696104743,1.7320307946172435,1.7348023242584145,1.7245699271907204],"val_acc":[0.035596707099933685,0.11646090521488661,0.1783950619184922,0.2372427979951778,0.2779835402781581,0.32181069978470667,0.3639917706020575,0.4162551437201814,0.4360082294837928,0.4473251023103671,0.4582304539074623,0.4726337429917889,0.49938271443049115,0.5055555545498804,0.5469135832762031,0.5452674895892908,0.5633744885646758,0.567489713737013,0.5609053516216239,0.5962962979151879,0.5707818932003446,0.5786008262707864,0.6195473312227814,0.593209880185716,0.6360082343649962],"val_loss":[4.329477495617336,3.663249999897961,2.7979632321699164,2.4209758465182145,2.058036261379964,1.9013591340049305,1.7624764768675032,1.5960869695914626,1.546456868756455,1.5112889467934032,1.457471612549613,1.4057871458952318,1.3824784856274295,1.3261019761670274,1.2932775035316562,1.25360278313052,1.2821995272557922,1.2447420937534222,1.227723549913477,1.2087008624410434,1.1869880206300398,1.1920295531857652,1.1072036170174555,1.1887371964415405,1.1634688541722396]},"selected":{"id":"17220","type":"Selection"},"selection_policy":{"id":"17221","type":"UnionRenderers"}},"id":"16750","type":"ColumnDataSource"},{"attributes":{"data_source":{"id":"16750","type":"ColumnDataSource"},"glyph":{"id":"16775","type":"Line"},"hover_glyph":null,"muted_glyph":null,"nonselection_glyph":{"id":"16776","type":"Line"},"selection_glyph":null,"view":{"id":"16778","type":"CDSView"}},"id":"16777","type":"GlyphRenderer"},{"attributes":{"axis_label":"Epoch","formatter":{"id":"17218","type":"BasicTickFormatter"},"ticker":{"id":"16798","type":"BasicTicker"}},"id":"16797","type":"LinearAxis"},{"attributes":{"active_drag":"auto","active_inspect":"auto","active_multi":null,"active_scroll":"auto","active_tap":"auto","tools":[{"id":"16751","type":"HoverTool"}]},"id":"16771","type":"Toolbar"},{"attributes":{"axis_label":"Loss","formatter":{"id":"17216","type":"BasicTickFormatter"},"ticker":{"id":"16803","type":"BasicTicker"}},"id":"16802","type":"LinearAxis"},{"attributes":{"active_drag":"auto","active_inspect":"auto","active_multi":null,"active_scroll":"auto","active_tap":"auto","tools":[{"id":"16787","type":"HoverTool"}]},"id":"16807","type":"Toolbar"},{"attributes":{},"id":"17214","type":"BasicTickFormatter"},{"attributes":{},"id":"16798","type":"BasicTicker"},{"attributes":{},"id":"17216","type":"BasicTickFormatter"},{"attributes":{"callback":null,"tooltips":[["Loss","@loss"],["Validation Loss","@val_loss"],["Epoch","@epoch"]]},"id":"16787","type":"HoverTool"},{"attributes":{"source":{"id":"16750","type":"ColumnDataSource"}},"id":"16783","type":"CDSView"},{"attributes":{"dimension":1,"ticker":{"id":"16803","type":"BasicTicker"}},"id":"16806","type":"Grid"},{"attributes":{"line_color":"orange","line_width":2,"x":{"field":"epoch"},"y":{"field":"val_acc"}},"id":"16780","type":"Line"},{"attributes":{},"id":"16803","type":"BasicTicker"},{"attributes":{},"id":"17220","type":"Selection"}],"root_ids":["16824"]},"title":"Bokeh Application","version":"1.3.4"}}
        </script>
        <script type="text/javascript">
          (function() {
            var fn = function() {
              Bokeh.safely(function() {
                (function(root) {
                  function embed_document(root) {

                  var docs_json = document.getElementById('17436').textContent;
                  var render_items = [{"docid":"9dc03127-455d-40c1-85b1-19d65f6ebbe3","roots":{"16824":"c0277cb6-8bc7-4251-84c5-84345b6cca14"}}];
                  root.Bokeh.embed.embed_items(docs_json, render_items);

                  }
                  if (root.Bokeh !== undefined) {
                    embed_document(root);
                  } else {
                    var attempts = 0;
                    var timer = setInterval(function(root) {
                      if (root.Bokeh !== undefined) {
                        embed_document(root);
                        clearInterval(timer);
                      }
                      attempts++;
                      if (attempts > 100) {
                        console.log("Bokeh: ERROR: Unable to run BokehJS code because BokehJS library is missing");
                        clearInterval(timer);
                      }
                    }, 10, root)
                  }
                })(window);
              });
            };
            if (document.readyState != "loading") fn();
            else document.addEventListener("DOMContentLoaded", fn);
          })();
        </script>

  </body>

</html>


![The accuracy and loss over epochs in phase 1]({{site.url}}{{site.baseurl}}/assets/images/BigBrainBeatv3_phase1_accAndLoss.png)
![The accuracy and loss over epochs in phase 2]({{site.url}}{{site.baseurl}}/assets/images/BigBrainBeatv3_phase2_accAndLoss.png)
![The accuracy and loss over epochs in phase 3]({{site.url}}{{site.baseurl}}/assets/images/BigBrainBeatv3_phase3_accAndLoss.png)

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
