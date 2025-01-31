# boltzmann

This library is supposed to implement [Boltzmann Machines](https://en.wikipedia.org/wiki/Boltzmann_machine), Autoencoders and related deep learning technologies. All implementations should both have a clean high-level mathematical implementation of their algorithms (with `core.matrix`) and if possible, an optimized and benchmarked version of the core routines for production use. This is to facilitate learning for new users or potential contributors, to be able to implement algorithms from papers/other languages and then tune them for performance if needed.

This repository is supposed to cover techniques building on [Restricted Boltzmann Machines](https://en.wikipedia.org/wiki/Restricted_Boltzmann_machine), like [Deep Belief Networks](https://en.wikipedia.org/wiki/Deep_belief_network), [Deep Boltzmann Machines](http://www.cs.toronto.edu/~fritz/absps/dbm.pdf) or temporal extensions thereof as well as Autoencoders (which I am not familiar enough with yet). Classical back-propagation is also often used to fine-tune deep models supervisedly, so networks should support it as well.

This is a state-of-the-art [reference for deep learning in general in Python](http://deeplearning.net/tutorial/). Feel free to port examples to this library as an exercise and open pull-requests :).

A reconstruction of the provided rbm vs. original digits:

![reconstruction](../master/resources/reconstructions.png)
![original](../master/resources/original.png)

## Usage

Add a depencency to your leiningen project:
[![Clojars Project](http://clojars.org/es.topiq/boltzmann/latest-version.svg)](http://clojars.org/es.topiq/boltzmann)

First you have to fetch the [mnist dataset](http://yann.lecun.com/exdb/mnist/) and put it (gunzipped) into `resources`.
~~~clojure
;; test the (fast) jblas implementation on mnist
user> (require '[boltzmann.mnist :as mnist]
           '[boltzmann.visualize :as v]
           '[boltzmann.core :refer :all]
           '[boltzmann.jblas :refer :all]
           '[clojure.core.matrix :refer [matrix] :as mat])
nil
;; ensure we are using clatrix (jblas)
user> (mat/current-implementation)
:clatrix
;; load the images, this can take up to a few minutes, so you can use a future
user> (future (def images (mnist/read-images "resources/train-images-idx3-ubyte"))
          (def batches (doall (map matrix (partition 10 images)))))
#<core$future_call$reify__6320@4c5681f1: :pending>
;; read label data (fast)
user> (def labels (mnist/read-labels "resources/train-labels-idx1-ubyte"))
#'user/labels
;; unify images and labels in one data vector
user> (def labeled-images
    (map #(float-array (concat %1 %2)) (map bin-label labels) images))
#'user/labeled-images
;; we use mini-batches of size 100 for a speedup
user> (def labeled-batches
    (doall (map matrix (partition 100 labeled-images))))
#'user/labeled-batches
;; train with 100 hidden units
user> (def trained
    (let [rbm (create-jblas-rbm 794 100)]
      (time (train-cd rbm labeled-batches :epochs 4 :init-learning-rate 0.01 :k 1))))
#'user/trained
;; build test-images with zeroed labels
user> (def test-images (map (comp float-array concat) (repeat (repeat 10 0)) images))
#'user/test-images
;; take a peek at the reconstructions of the first 100 images
user> (->> test-images
       (take 100)
       (map (comp matrix vector))
       (map #(->> %
                  (probs-hs-given-vs [(:restricted-weights trained)
                                      (:h-biases trained)])
                  (probs-vs-given-hs [(:restricted-weights trained)
                                      (:v-biases trained)])))
       (map (partial drop 10))
       (map #(partition 28 %))
       (v/tile 10)
       v/render-grayscale-float-matrix
       v/view)
#<JFrame javax.swing.JFrame[frame0,10,62,280x280,layout=java.awt.BorderLayout,title=MNIST Digit,normal,defaultCloseOperation=HIDE_ON_CLOSE,rootPane=javax.swing.JRootPane[,0,0,280x280,layout=javax.swing.JRootPane$RootLayout,alignmentX=0.0,alignmentY=0.0,border=,flags=16777673,maximumSize=,minimumSize=,preferredSize=],rootPaneCheckingEnabled=true]>
;; classifier the first 10% of training data
user> (def classified
    (->> test-images
         (take 6000)
         (map (comp matrix vector))
         (map #(->> %
                    (probs-hs-given-vs [(:restricted-weights trained)
                                        (:h-biases trained)])
                    (probs-vs-given-hs [(:restricted-weights trained)
                                        (:v-biases trained)])))
         (map (partial take 10))
         (map max-index)))
#'user/classified
;; check how good we model our training data (validation data gives better approximation)
user> (float (classification-rate labels classified))
0.697
;; 70% of the labels with the highest probability are correct
;; now let's load a pretrained rbm with 500 hidden units, trained for 40 epochs with CD-1
user> (require '[clojure.edn :as edn])
nil
user> (def trained
    (edn/read-string {:readers {'boltzmann.jblas/JBlasRBM load-jblas-rbm}}
                     (slurp "resources/mnist-labeled.edn")))
#'user/trained
;; the reconstructions are much better and fairly sharp
user> (->> test-images
       (take 100)
       (map (comp matrix vector))
       (map #(->> %
                  (probs-hs-given-vs [(:restricted-weights trained)
                                      (:h-biases trained)])
                  (probs-vs-given-hs [(:restricted-weights trained)
                                      (:v-biases trained)])))
       (map (partial drop 10))
       (map #(partition 28 %))
       (v/tile 10)
       v/render-grayscale-float-matrix
       v/view)
#<JFrame javax.swing.JFrame[frame2,0,24,280x280,layout=java.awt.BorderLayout,title=MNIST Digit,normal,defaultCloseOperation=HIDE_ON_CLOSE,rootPane=javax.swing.JRootPane[,0,0,280x280,layout=javax.swing.JRootPane$RootLayout,alignmentX=0.0,alignmentY=0.0,border=,flags=16777673,maximumSize=,minimumSize=,preferredSize=],rootPaneCheckingEnabled=true]>
;; reclassify
user> (def classified
    (->> test-images
         (take 6000)
         (map (comp matrix vector))
         (map #(->> %
                    (probs-hs-given-vs [(:restricted-weights trained)
                                        (:h-biases trained)])
                    (probs-vs-given-hs [(:restricted-weights trained)
                                        (:v-biases trained)])))
         (map (partial take 10))
         (map max-index)))
#'user/classified
;; now we get 93.6% of the labels right
user> (float (classification-rate labels classified))
0.93583333
~~~

You can find more examples including bar-charts of the theoretical (naive) implementation in the comments of `core.clj` and `jblas.clj`.

## Changelog

### 0.1.2
- make learning rate tunable
- keep async history of training to evaluate process
- improve gibbs sampling
- bump deps

### 0.1.1
- expose seeds where possible (not yet in incanter(?))

### 0.1
- initial RBM version
- CD implementation (slow and fast)
- mnist loading
- drawing of receptive fields
- classification error

## TODO
- have a look at https://github.com/jcsims/deebn and synaptic and merge
- fix random seed behaviour, so runs become reproducible
- remove blobs from jar
- find replacement for incanter's non-seedable sample-normal fn
- track reconstruction error asynchronously and allow it to control/finish training
- factor out mnist (norb etc.) data set learning for others
- implement PCD
- implement more plots & visualize training of mnist
- introduce fixed seeds?
- do benchmarking (esp. of matrix ops) and algorithmic optimizations:
  investigate https://github.com/fommil/netlib-java/ with mtj, for 1000x1000 matrices 10x speedup on cpu vs. jBLAS on google mmul benchmark (jBLAS has support for netlib as well); investigate jcublas for gpu
- implement/port AIS, CAST, AST sampling
- make incanter dev dependency
- try to hide from skynet ;-)

## License

Copyright © 2014-2015 Christian Weilbach

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
