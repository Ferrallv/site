---
title: "BigBrainBeat - A neural network to detect BPM"
excerpt_separator: "<!--more-->"
categories:
  - Project
---



___

## *TL;DR*

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

#### The accuracy and loss over epochs in phase 1 and 2

{% include BigBrainBeatv3_phase1_accAndLoss.html %}

{% include BigBrainBeatv3_phase2_accAndLoss.html %}

<html lang="en">
  
  <head>
    
      <meta charset="utf-8">
      <title>Bokeh Plot</title>
      
      
        
          
        
        
          
        <script type="text/javascript" src="https://cdn.pydata.org/bokeh/release/bokeh-1.3.4.min.js"></script>
        <script type="text/javascript">
            Bokeh.set_log_level("info");
        </script>
        
      
      
    
  </head>
  
  
  <body>
    
      
        
          
          
            
              <div class="bk-root" id="19b19ff3-1a16-4427-9d5b-e144359f16cc" data-root-id="1075"></div>
            
          
        
      
      
        <script type="application/json" id="1290">
          {"e00b996a-9909-40a0-8262-192a1e36fe2e":{"roots":{"references":[{"attributes":{},"id":"1089","type":"Selection"},{"attributes":{"data_source":{"id":"1001","type":"ColumnDataSource"},"glyph":{"id":"1067","type":"Line"},"hover_glyph":null,"muted_glyph":null,"nonselection_glyph":{"id":"1068","type":"Line"},"selection_glyph":null,"view":{"id":"1070","type":"CDSView"}},"id":"1069","type":"GlyphRenderer"},{"attributes":{},"id":"1018","type":"BasicTicker"},{"attributes":{"callback":null},"id":"1040","type":"DataRange1d"},{"attributes":{"dimension":1,"ticker":{"id":"1018","type":"BasicTicker"}},"id":"1021","type":"Grid"},{"attributes":{"source":{"id":"1001","type":"ColumnDataSource"}},"id":"1070","type":"CDSView"},{"attributes":{"callback":null},"id":"1042","type":"DataRange1d"},{"attributes":{"line_color":"blue","line_width":2,"x":{"field":"epoch"},"y":{"field":"acc"}},"id":"1026","type":"Line"},{"attributes":{"callback":null,"data":{"acc":[0.03640350828502785,0.11166666655621507,0.16394736825308778,0.22456140329309723,0.28842105285117503,0.3307017555231588,0.36140350932091997,0.3914912288779752,0.4154385958324399,0.47236842047749905,0.5063157886789557,0.531052630852189,0.5562280711897633,0.5857894761781943,0.6119298274841225,0.6367543920090324,0.6361403567226309,0.6728947414902219,0.6746491274812765,0.6884210586809275,0.6972807068050954,0.7048245638347509,0.7208771940908933,0.729912283127768,0.7387719277227134,0.7506140355478254,0.7614035062622606,0.7679824565063443,0.7745614025676459,0.780526311512579,0.7864035064713997,0.8015789441895067,0.7988596438315877,0.8084210481560021,0.8057017491052025],"epoch":[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35],"loss":[4.424929407604954,3.6204288852842232,3.018990797954693,2.6851512381905005,2.424537254007239,2.2222168863865366,2.096156608640102,1.9833819390388958,1.8920122165428965,1.6909070793996777,1.5032820549973271,1.422767145069022,1.329100268713215,1.238037805546794,1.1736792707652377,1.113910889677834,1.0981068292207885,1.0248054739153176,0.9895373980204264,0.9534576205830825,0.9269966343254373,0.9193356374376699,0.8695446974352786,0.8426420240287196,0.8089010536409261,0.7874827677743477,0.7395998149325973,0.7166867088853267,0.7052685164830141,0.6851547236243883,0.6682902032084632,0.6279146764100644,0.6152917317261821,0.5980107554871785,0.6018900679130303],"val_acc":[0.1013157886305922,0.23934210577097378,0.29921052732357856,0.4494736830850965,0.520789474444954,0.5618421072630506,0.604210527711793,0.6221052668988705,0.6622368444345499,0.7030263213734878,0.7305263178913217,0.7317105241511997,0.7815789431333542,0.7963157856934949,0.8149999952629993,0.8281578915683847,0.8486842046442785,0.8538157834034217,0.875921049008244,0.8538157845798292,0.8821052595188743,0.876710521155282,0.8760526262615856,0.8907894703902697,0.8893421009967202,0.8894736798186051,0.8792105207317754,0.8971052577621058,0.8919736790029626,0.8993421028319158,0.8961842040482321,0.9124999963923505,0.9182894700451901,0.9124999960002146,0.9064473635271976],"val_loss":[3.9176038456590554,2.7882580898310008,2.2503346573365364,1.889926123775934,1.7309815326803608,1.6298496550635289,1.5108636821571149,1.4039869430033785,1.2829571013387882,1.0683233239933063,0.9577533979164926,0.8581547454783791,0.7949837940303903,0.7361884809246189,0.650462484477382,0.6288550603938731,0.6018892197232497,0.5585442101092715,0.5225810833079251,0.522950155758544,0.4780313237325141,0.4841054576007943,0.4472669537522291,0.4085011154805359,0.3910055455604666,0.39502184465527534,0.38789754605999116,0.3512402025884704,0.3487939966940566,0.34141228416640507,0.3517728800836362,0.2914328888842934,0.2827451220272403,0.28440853139679684,0.29074778569568144]},"selected":{"id":"1089","type":"Selection"},"selection_policy":{"id":"1088","type":"UnionRenderers"}},"id":"1001","type":"ColumnDataSource"},{"attributes":{},"id":"1044","type":"LinearScale"},{"attributes":{"index":1,"label":{"value":"Validation Accuracy"},"renderers":[{"id":"1069","type":"GlyphRenderer"}]},"id":"1072","type":"LegendItem"},{"attributes":{"active_drag":"auto","active_inspect":"auto","active_multi":null,"active_scroll":"auto","active_tap":"auto","tools":[{"id":"1002","type":"HoverTool"}]},"id":"1022","type":"Toolbar"},{"attributes":{"text":"Accuracy by Epoch for BigBrainBeatv3_phase3"},"id":"1024","type":"Title"},{"attributes":{},"id":"1046","type":"LinearScale"},{"attributes":{"items":[{"id":"1071","type":"LegendItem"},{"id":"1072","type":"LegendItem"}]},"id":"1073","type":"Legend"},{"attributes":{},"id":"1088","type":"UnionRenderers"},{"attributes":{"axis_label":"Epoch","formatter":{"id":"1086","type":"BasicTickFormatter"},"ticker":{"id":"1049","type":"BasicTicker"}},"id":"1048","type":"LinearAxis"},{"attributes":{"line_alpha":0.1,"line_color":"#1f77b4","line_width":2,"x":{"field":"epoch"},"y":{"field":"val_loss"}},"id":"1068","type":"Line"},{"attributes":{"line_alpha":0.1,"line_color":"#1f77b4","line_width":2,"x":{"field":"epoch"},"y":{"field":"acc"}},"id":"1027","type":"Line"},{"attributes":{},"id":"1049","type":"BasicTicker"},{"attributes":{"data_source":{"id":"1001","type":"ColumnDataSource"},"glyph":{"id":"1026","type":"Line"},"hover_glyph":null,"muted_glyph":null,"nonselection_glyph":{"id":"1027","type":"Line"},"selection_glyph":null,"view":{"id":"1029","type":"CDSView"}},"id":"1028","type":"GlyphRenderer"},{"attributes":{"ticker":{"id":"1049","type":"BasicTicker"}},"id":"1052","type":"Grid"},{"attributes":{"children":[{"id":"1003","subtype":"Figure","type":"Plot"},{"id":"1039","subtype":"Figure","type":"Plot"}]},"id":"1075","type":"Column"},{"attributes":{"source":{"id":"1001","type":"ColumnDataSource"}},"id":"1029","type":"CDSView"},{"attributes":{"axis_label":"Loss","formatter":{"id":"1084","type":"BasicTickFormatter"},"ticker":{"id":"1054","type":"BasicTicker"}},"id":"1053","type":"LinearAxis"},{"attributes":{"index":0,"label":{"value":"Training Accuracy"},"renderers":[{"id":"1028","type":"GlyphRenderer"}]},"id":"1035","type":"LegendItem"},{"attributes":{"below":[{"id":"1012","type":"LinearAxis"}],"center":[{"id":"1016","type":"Grid"},{"id":"1021","type":"Grid"},{"id":"1037","type":"Legend"}],"left":[{"id":"1017","type":"LinearAxis"}],"plot_height":300,"plot_width":500,"renderers":[{"id":"1028","type":"GlyphRenderer"},{"id":"1033","type":"GlyphRenderer"}],"title":{"id":"1024","type":"Title"},"toolbar":{"id":"1022","type":"Toolbar"},"x_range":{"id":"1004","type":"DataRange1d"},"x_scale":{"id":"1008","type":"LinearScale"},"y_range":{"id":"1006","type":"DataRange1d"},"y_scale":{"id":"1010","type":"LinearScale"}},"id":"1003","subtype":"Figure","type":"Plot"},{"attributes":{"active_drag":"auto","active_inspect":"auto","active_multi":null,"active_scroll":"auto","active_tap":"auto","tools":[{"id":"1038","type":"HoverTool"}]},"id":"1058","type":"Toolbar"},{"attributes":{},"id":"1080","type":"BasicTickFormatter"},{"attributes":{},"id":"1054","type":"BasicTicker"},{"attributes":{"line_color":"orange","line_width":2,"x":{"field":"epoch"},"y":{"field":"val_acc"}},"id":"1031","type":"Line"},{"attributes":{"dimension":1,"ticker":{"id":"1054","type":"BasicTicker"}},"id":"1057","type":"Grid"},{"attributes":{"callback":null},"id":"1004","type":"DataRange1d"},{"attributes":{"line_alpha":0.1,"line_color":"#1f77b4","line_width":2,"x":{"field":"epoch"},"y":{"field":"val_acc"}},"id":"1032","type":"Line"},{"attributes":{"callback":null},"id":"1006","type":"DataRange1d"},{"attributes":{"line_color":"blue","line_width":2,"x":{"field":"epoch"},"y":{"field":"loss"}},"id":"1062","type":"Line"},{"attributes":{"data_source":{"id":"1001","type":"ColumnDataSource"},"glyph":{"id":"1031","type":"Line"},"hover_glyph":null,"muted_glyph":null,"nonselection_glyph":{"id":"1032","type":"Line"},"selection_glyph":null,"view":{"id":"1034","type":"CDSView"}},"id":"1033","type":"GlyphRenderer"},{"attributes":{},"id":"1082","type":"BasicTickFormatter"},{"attributes":{},"id":"1008","type":"LinearScale"},{"attributes":{"callback":null,"tooltips":[["Validation Accuracy","@val_acc"],["Accuracy","@acc"],["Epoch","@epoch"]]},"id":"1002","type":"HoverTool"},{"attributes":{"source":{"id":"1001","type":"ColumnDataSource"}},"id":"1034","type":"CDSView"},{"attributes":{"line_alpha":0.1,"line_color":"#1f77b4","line_width":2,"x":{"field":"epoch"},"y":{"field":"loss"}},"id":"1063","type":"Line"},{"attributes":{},"id":"1084","type":"BasicTickFormatter"},{"attributes":{},"id":"1010","type":"LinearScale"},{"attributes":{"data_source":{"id":"1001","type":"ColumnDataSource"},"glyph":{"id":"1062","type":"Line"},"hover_glyph":null,"muted_glyph":null,"nonselection_glyph":{"id":"1063","type":"Line"},"selection_glyph":null,"view":{"id":"1065","type":"CDSView"}},"id":"1064","type":"GlyphRenderer"},{"attributes":{"text":"Loss by Epoch for BigBrainBeatv3_phase3"},"id":"1060","type":"Title"},{"attributes":{},"id":"1086","type":"BasicTickFormatter"},{"attributes":{"axis_label":"Epoch","formatter":{"id":"1082","type":"BasicTickFormatter"},"ticker":{"id":"1013","type":"BasicTicker"}},"id":"1012","type":"LinearAxis"},{"attributes":{"source":{"id":"1001","type":"ColumnDataSource"}},"id":"1065","type":"CDSView"},{"attributes":{"index":1,"label":{"value":"Validation Accuracy"},"renderers":[{"id":"1033","type":"GlyphRenderer"}]},"id":"1036","type":"LegendItem"},{"attributes":{},"id":"1013","type":"BasicTicker"},{"attributes":{"index":0,"label":{"value":"Training Accuracy"},"renderers":[{"id":"1064","type":"GlyphRenderer"}]},"id":"1071","type":"LegendItem"},{"attributes":{"items":[{"id":"1035","type":"LegendItem"},{"id":"1036","type":"LegendItem"}],"location":"bottom_right"},"id":"1037","type":"Legend"},{"attributes":{"ticker":{"id":"1013","type":"BasicTicker"}},"id":"1016","type":"Grid"},{"attributes":{"line_color":"orange","line_width":2,"x":{"field":"epoch"},"y":{"field":"val_loss"}},"id":"1067","type":"Line"},{"attributes":{"callback":null,"tooltips":[["Loss","@loss"],["Validation Loss","@val_loss"],["Epoch","@epoch"]]},"id":"1038","type":"HoverTool"},{"attributes":{"axis_label":"Accuracy","formatter":{"id":"1080","type":"BasicTickFormatter"},"ticker":{"id":"1018","type":"BasicTicker"}},"id":"1017","type":"LinearAxis"},{"attributes":{"below":[{"id":"1048","type":"LinearAxis"}],"center":[{"id":"1052","type":"Grid"},{"id":"1057","type":"Grid"},{"id":"1073","type":"Legend"}],"left":[{"id":"1053","type":"LinearAxis"}],"plot_height":300,"plot_width":500,"renderers":[{"id":"1064","type":"GlyphRenderer"},{"id":"1069","type":"GlyphRenderer"}],"title":{"id":"1060","type":"Title"},"toolbar":{"id":"1058","type":"Toolbar"},"x_range":{"id":"1040","type":"DataRange1d"},"x_scale":{"id":"1044","type":"LinearScale"},"y_range":{"id":"1042","type":"DataRange1d"},"y_scale":{"id":"1046","type":"LinearScale"}},"id":"1039","subtype":"Figure","type":"Plot"}],"root_ids":["1075"]},"title":"Bokeh Application","version":"1.3.4"}}
        </script>
        <script type="text/javascript">
          (function() {
            var fn = function() {
              Bokeh.safely(function() {
                (function(root) {
                  function embed_document(root) {
                    
                  var docs_json = document.getElementById('1290').textContent;
                  var render_items = [{"docid":"e00b996a-9909-40a0-8262-192a1e36fe2e","roots":{"1075":"19b19ff3-1a16-4427-9d5b-e144359f16cc"}}];
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


<a name="fn1">1</a>: Bouwer FL, Van Zuijen TL, Honing H (2014) Beat Processing Is Pre-Attentive for Metrically Simple Rhythms with Clear Accents: An ERP Study. PLoS ONE 9(5): e97467. https://doi.org/10.1371/journal.pone.0097467
