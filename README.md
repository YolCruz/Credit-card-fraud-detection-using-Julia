# Credit-card-fraud-detection-using-Julia

This simple project takes the credit card fraud detection dataset, available at [kaggle](https://www.kaggle.com/mlg-ulb/creditcardfraud), and uses the Flux library to predict which transaction is fraudulent and which is not.

## Divide the data

Since the dataset is a single csv file, a division has to be made in order to have a separate set of data in which test the final model:

```
using CSV

dataset = CSV.read("dataset/creditcard.csv")
data_size = length(dataset[:,1])
284807

test_data_size = round(Int, data_size*0.2)
56961

train_data = dataset[1:end-test_data_size, :]
test_data = dataset[end-test_data_size+1:end, :]

CSV.write("dataset/train_data.csv", train_data)
CSV.write("dataset/test_data.csv", test_data)
```

Now we have two datasets, the train set "train_data", and the test set "test_data".

## Analising the dataset

We already know that this dataset is highly imbalanced. If we run the fuction ```neg_pos(file)``` defined in *datapreprocessing.jl*, we can know how many negative and positive labels are in the dataset. We'll say that the data is labeled negative if it's column named Class has a zero, which means that the transaction was not fraudulent.

```
const file = "dataset/train_data.csv"
neg, pos = neg_pos(file)
(227429, 417)
```

The fraudulent translactions (positive labels) constitudes only 18.3% of the training set. This means that training normaly like with any other dataset will not help to make good predictions, because any model can learn to always predict a negative label, getting a high accuracy. 

To make the model learn more from the positive label, I choose to change the objective function, so the model gets a high loss when it encounters a positive label:

```
function loss_function(x, y)
    if y != 0
        loss = binarycrossentropy(model(x)[1], y) .* 100
    else
        loss = binarycrossentropy(model(x)[1], y) .* 1
    end
    return loss
end
```

## Making the pipeline

Since there is a lot of data, it would be memory intensive to load it all and pass it to the model. Because of this, I made a function to load the data in batches, with the option to shuffle it. Shuffling the data is very important in order to avoid overfitting the model:

```
using CuArrays, CSV
import Random: shuffle

function load_data(path, batch_size, start; _shuffle=true)
    x = []
    y = []
    pairs = []
    for row in CSV.Rows(path, skipto=start, limit=batch_size)
        rows = [row.V1, row.V2, row.V3, row.V4, row.V5, row.V6, row.V7, row.V8, row.V9,
                row.V10, row.V11, row.V12, row.V13, row.V14, row.V15, row.V16, row.V17,
                row.V18, row.V19, row.V20, row.V21, row.V22, row.V23, row.V24, row.V25,
                row.V26, row.V27, row.V28, log(parse(Float32, row.Amount)+0.0001)]
        rows = string_to_number(rows)
        push!(pairs, rows => parse(Float32, row.Class))
    end

    if _shuffle
        pairs = shuffle(pairs)
    end

    for pair in pairs
        push!(x, pair.first)
        push!(y, pair.second)
    end
    return gpu(gpu.(x)), gpu(gpu.(y))
end
```
Notice how when parsing the column Amount I apply the fuction Log and sum a little quantity. This is done because that column has a range much greater than the other 28 columns, and the log fuction reduces that number. I add 0.0001 to never  have a zero or NaN Amount.

The important method of this function is CSV.Rows, which reads a CSV file row by row rather than the entire file. The arguments *batch_size* and *start* determine how many rows will be read and from which one. These are important as they will help pipelinening the training. The fuction ```string_to_number(rows)``` is a simple handling function that takes arrays with string elements as input and returns an array of numbers: 

```
function string_to_number(x::AbstractArray)
    placeholder = []
    for item in x
        if typeof(item) == String
            push!(placeholder, parse(Float32, item))
        else
            push!(placeholder, item)
        end
    end
    return placeholder
end
```
The package CuArrays is used to move the arrays containing the data to the gpu, for a much more faster training.

## Creating the model

The model will only have two Dense layers and a Dropout layer, since the dataset is already normalized.

```
model = Chain(
        Dense(29, 64, relu),
        Dropout(0.5),
        Dense(64, 1, sigmoid)
) |> gpu
```

With the wraper function ```|> gpu``` the model is moved to the gpu.

Training several times, we see that the weigths quickly decay to NaN. Because of this the optimiser used will be a combination of inverse decay and ADAM.

```
opt = Flux.Optimiser(InvDecay(), ADAM())
```

## Evaluating the model

Since the dataset is imbalanced, we need to use an accuracy metric different than a confuse matrix. AUC is a good metric. The file utils.jl provides several methods to use this metric, using simple input arrays of zeros and ones.
After only 23 batches, training with a batch size of 2048, to ensure that every batch has at least one positive label, the accuracy reached with the test set is of 0.9953 or **99.53%**. The next two plots show the loss and the accuracy during the training.

![Total loss for the batch at time_step.](/plots/loss_plot.png)
![Total accuracy for the batch at time_step.](/plots/accuracy_plot.png)

The training was very quick, but we can see that the model rapidly learned how to predict correctly a positive label.
