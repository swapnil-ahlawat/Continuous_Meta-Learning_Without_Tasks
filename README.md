# Continuous_Meta-Learning_Without_Tasks

Notebook by: Swapnil Ahlawat

Paper link: https://arxiv.org/pdf/1912.08866.pdf

## Setup Instructions
Run the following command to install all the relevant packages:

`pip3 install torch torchvision torchaudio && pip install matplotlib && pip install scikit-learn &&
pip install learn2learn`

In case the installation of learn2learn throws error, run the following command and pip install
learn2learn again:

`sudo apt install libjpeg8-dev zlib1g-dev`

The notebook contains all the relevant code to generate the required datasets, run models and
get inferences. For faster reproducibility, all the weights and datasets used are saved and given
with this notebook. To reproduce the result, you can directly move on to the result/test accuracy
section of each model and get the results (without training the model).
Running the complete notebook will overwrite these weights and datasets, so kindly keep a
backup.

## Folder Contents
1. MOCA.ipynb: jupyter notebook with all the relevant code.
2. datasets/: Datasets for classification and time series forecasting tasks generated and stored
here.
3. weights/: Weights of all the models used stored to replicate the results.
4. results/: forecasting video, accuracy graph, etc.

## Brief Summary of Paper
The paper tries to develop a generic wrapping algorithm that can enable any meta learning
algorithm to perform well on unsegmented time-series data. It presents meta-learning via online
changepoint analysis (MOCA), an approach that combines meta learning with differentiable
Bayesian changepoint detection schemes to softly detect task switches and make predictions
according to that.

For the prediction at every time step, MOCA maintains a belief (probability), b[r=t], over all
possible run-lengths (Here run lengths mean number of past data points belonging to current
task). It also maintains separate versions of the base meta learning algorithm’s posterior
parameters for all possible run-lengths, n[t][r=x]. Then it uses the belief probability along with
their probabilistic prediction for the label, p(yhat | x, n[t][r=x]), to calculate the final prediction
probability, p(yhat | x[1:t], y[1:t-1]). The belief is updated after observing every datapoint. This
way of maintaining belief acts as a soft version of estimating the task switch points in the
unsegmented time-series.

Putting all the MOCA steps mathematically:
1. Predict p(yhat | x[1:t], y[1:t−1]): Summation of b[r]*p(yhat | x[t], n[t-1][r] over all run
lengths.
2. Observe label for current time step and calculate loss.
3. Update posterior parameters: Calculate n[t][r] for all run-lengths
4. Compute b[r | x[t], y[t]]: proportional to b[r]*p(y[t] | x[t], n[t-1][r])
5. Update belief: If r=0, b[r]= Hazard rate, else b[r]= (1- Hazard rate)*b[r-1 | x[t], y[t]]

Hazard rate is probability that the current data point is the task switch point (Hyperparameter)
During training, the losses are cumulated over the complete time series and then gradient
update step is performed after that. There is no gradient update step in testing.
The paper demonstrates the utility of this approach on three nonlinear meta-regression
benchmarks as well as two meta-image-classification benchmarks.

## Experiment 1: Classification using MOCA with CNP
### Intuition behind using CNP
MOCA is required to maintain different versions of posterior parameters for various run-lengths
which serves as distribution representation for these run-lengths. In the paper, for the
classification task, they have assumed the data to follow Categorical-Gaussian generative
model and used Dirichlet prior over the class probabilities and a Gaussian prior over the mean
for each class and use its variables as posterior parameters.

Instead of following this complex probabilistic approach for classification, I have used CNP to
learn these posterior parameters and perform classification. For each run-length, the adapt step
in CNP encodes all the data points of that run-length into a vector that act as a posterior
parameter in MOCA(representation of distribution for this run-length). As the encoding weights
would also be updated in every training step, CNP would keep on learning a better way to
encode the distribution.

### Dataset:
A 20-way 20-dimensional euclidean dataset is used to generate the dataset for this
experiment. A long segmented time-series for training is generated by concatenating 1000
tasks. The number of shots in each task vary between 2 to 10. The same method is used to
generate time-series for testing but with only 100 tasks.

### Models
#### MOCA Models
As the whole data series is very long and impractical to train on, we'll sample short random
series from the whole series in every epoch and train on that.
##### Training parameters:
* Time-series length: 100-300 data points
* Epochs: 2000
* Learning rate: 1e-3 for first 1000 epochs and 1e-4 for next 1000 epochs
* Optimizer: Adam
* Hazard rate: [0.01, 0.05, 0.1, 0.25, 0.5] (Trained 5 different models)

The series lengths are kept considerably long (between 100-300 data points) to achieve good
performance on low hazard rates, as short batch lengths artificially increase the hazard rate as
a result of the assumption that each batch begins with a new task.

#### Ablation Performed: MOCA with Uniform Belief
To look at the importance of belief in MOCA, I've implemented another MOCA class, but in this
version, the belief remains Uniform for all run lengths (1/number of run-lengths).
##### Training parameters:
Same as that of MOCA models
#### Baseline model: Train on Everything
This implementation consists of ignoring task variation and treating the whole training time
series as one task. For this, only CNP is used and it is adapted on all past data points.
##### Training parameters:
Same as MOCA except learning rate is 1e-3 for whole 2000 epochs
#### Oracle Model: CNP with Task Switches known
To get the maximum achievable accuracy, we train an Oracle model. Here, the task switches of
the dataset are known beforehand and CNP is adapted on past data points of only that task.
##### Training parameters:
Same as MOCA except learning rate is 1e-3 for whole 2000 epochs

### Experiment Results
The models were tested on unsegmented time-series data with length varying from 50 to 300.
Following are the test accuracies of all models:

| Models  | Test accuracy |
| ------------- | ------------- |
| MOCA, Hazard rate: 0.01 | 79.56% |
| MOCA, Hazard rate: 0.05 | 80.17% |
| MOCA, Hazard rate: 0.10 | 82.43% |
| MOCA, Hazard rate: 0.25 | 83.04% |
| MOCA, Hazard rate: 0.50 | 79.57% |
| MOCA with Uniform Belief | 67.65% |
| Train on Everything Model | 55.56% |
| Oracle Model | 93.71% |

### Discussion and Conclusions
1. MOCA model with hazard rate 0.25 performs the best. The increased hazard rate is the
result of the assumption that each batch begins with a new task.
2. The ablated model, MOCA with Uniform Belief, performs significantly better than the
baseline Train on everything model. (12% improvement)
3. The accuracy of the ablated model, MOCA with Uniform Belief, is very less compared to
the complete MOCA model. Hence, the belief probability plays a very significant role in
MOCA. (15.5% improvement)
4. MOCA's accuracy is approximately 10% less than Oracle model's accuracy. The reason
behind this is a very high task switch rate in the dataset (Task switches after every 12
data points on an average). MOCA is an approximation of our Oracle model and it'll
generally waste at least 1 data point in getting to know that the task has switched.
MOCA's performance will keep on getting closer to Oracle model's performance with the
reduction of task switch rate.

## Experiment 2 (Extension of Paper): Time Series Forecasting using LSTM Meta-learner + MOCA
Implementation of MOCA on top of LSTM meta-learner and comparing its performance on 2
bouncing balls dataset as a forecasting task.

### Extensions performed and their reason/intuition
1. The paper only discussed the performance of MOCA on regression and classification
tasks. In this experiment, we'll be looking at MOCA's performance on a time-series
forecasting task. From the previous experiment, we know that MOCA can detect task
switches, but the motivation behind this experiment is that it can predict task switches.
2. MOCA requires base meta learning algorithm's conditional posterior predictive, p(y | x,
n[t][r=x]) to update its belief probabilities. But this is not directly available in regression
tasks. The paper uses a separate output from the decoder and calculations on that to
get this. Instead of doing that, I have attempted a much simpler way to calculate it by just
using mean squared error.

### Selection of Current Model/Dataset
Before finalising the current implemented model, 2 other approaches were implemented but
they couldn’t beat the current one. Following are the other approached that were tried:
1. As the approximate range of label y is between -0.2 to 0.2, tanh was used as the last
layer for the decoder in LSTM meta-learner as it will force the model to avoid sudden
changes in consecutive frames. But this dampened the movement of balls by a lot and
the simulation wasn’t very natural.
2. Using X[t+1] as the label for X[t]: this approach seemed like a natural way to perform
time-series forecasting, but the naturality in the simulation of this approach was worse
than that of the current approach. One explanation for this can be as we are scaling
down X to feed into the network, the labels (next timestep X) is also getting scaled down.
Whereas in the current approach, even when we are scaling down X, the label y remains
unscaled.

### Dataset
Because of the collisions against the wall/other balls, the bouncing balls dataset naturally has
task switches. But the difference here is that, unlike our previous dataset where there were no
relation between 2 tasks, these task switches can be predicted if the model learns to forecast
the ball's trajectory and what action to take on collision.

For training dataset, 250 simulations of 2 balls, 100 time length are generated, X. Labels are
generated as a coordinate difference between 2 consecutive frames, y[t]= X[t+1]- X[t]. The input,
X, is then scaled down to 0-1 range, as that range is more amenable for training with standard
network initialization.

### Models
LSTM meta-learner has been used as the base meta-learning algorithm for this experiment. In
this class, the LSTM input is the concatenation of the current encoded input z[t] = φ(x[t], w)
(encoded using MLP) and the label from the past timestep y[t−1]. In this way, through the LSTM
update process, the hidden state can process a sequence of input/label pairs and encode
statistics of the posterior distribution in the hidden state. The LSTM output and the encoded
input are then concatenated and feeded into a decoding network (MLP) which outputs the label
y[t]. By including z[t] as input to the decoder, we lessen the information that needs to be stored
in the hidden state.

#### MOCA with LSTM meta-learner
For using MOCA with LSTM learner, yhat is calculated for all run-lengths by feeding them one
by one to LSTM meta-learner but with different number of past data points. The final regression
prediction yhat is the weighted average of yhat from all run lengths, weighted by the belief of
that run-length.
##### Calculating p(y[t] | x[t], n[t][r=x]) in regression setting:
For easier and more intuitional calculation, p(y[t] | x[t], n[t][r=x]) is calculated by first getting the
MSE loss individually on predictions from all run-lengths and then taking the inverse of it. As the
predicted coordinates for a particular run-length digress from the label, MSE loss increases and
its inverses decreases, leading to drop in belief probability for that run-length.
##### Training Parameters:
* Epochs: 750 (Model was overfitting post 750 epochs)
* Batch size: 8
* Learning rate: 1e-3 for first 500 epochs and 1e-4 for next 250 epochs
* Optimizer: Adam
* Encoding MLP dimensions: [4, 8, 16, 8]
* LSTM dimensions: 64-dimensional single layer
* Decoding MLP dimensions: [72 (64+8), 128, 4]
* Hazard rate: 0.01

#### Baseline Model: Condition on Everything
This model only uses LSTM meta-learner which runs on all available past data, as the LSTM
can theoretically learn to only consider relevant data.
##### Training parameters:
Training parameters same as that of MOCA except epochs and learning rate.
* Epochs: 1000
* Learning rate: 1e-3 for first 500 epochs and 1e-4 for next 500 epochs

### Experiment Results
The models were tested on 32 simulations of length 100 frames out of which they were feeded
only the first 10 frames and had to predict the next 90 frames.
Following are the results:
* MOCA with LSTM Learner Loss: 0.067
* Baseline Model Loss: 0.082
### Discussion and Conclusion
1. MSE loss for MOCA with LSTM meta-learner is significantly less than baseline model on
the time-series forecasting task. Even though it is only 0.015, the label y values stays in
the approximate range of [-1.5, 1.5] and 0.015 loss difference means an average
deviation of 0.122, which is significant. Hence, MOCA positively helps in predicting task
switches and making right predictions.
2. The simple conditional posterior probability calculation using MSE loss reduced the
complexity of the model while not negatively impacting the performance.
3. On visually comparing the forecasting of MOCA and the baseline model, we can see that
even though both the able have not been able to accurately predict trajectories and
collisions, MOCA is able to capture the effect of gravity and bouncing off the ground
better than the baseline model. The prediction of MOCA seems more naturally plausible
than that of the baseline model.
