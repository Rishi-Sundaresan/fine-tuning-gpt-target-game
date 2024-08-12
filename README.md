# Fine Tuning GPT 3.5 for 24-style Math Puzzles
## Summary
I fine-tuned GPT 3.5-turbo to be better at a math puzzle game, specifically a 24-style game. In this game, you are given 4 numbers, and you must add/subtract/multiply/divide between them to reach a target number. You must use all the numbers.

Example:
```
Given [5,11,11,9], make 36:
Answer: (5 - 11 / 11) * 9 = 36
```

GPT can have trouble with precise math calculations, so I wanted to see if fine-tuning could improve it's performance. Overall, **our custom model outperforms GPT 3-5-turbo-1106 with a ~ 3x higher accuracy on these problems**.
<br><br>
## Training
### Data Curation
I used a target number of 36 instead of 24 since it had a larger problem space. I generated hundreds of different examples of 4 numbers being used to hit 36. I constructed each training example for the fine-tuning job as follows:

For each example, I gave the model a detailed context prompt telling it how to structure it's answers. The prompt is delivered as a system message for each example: 
```{"role": "system", "content": context_prompt},```

````
You are playing a game where you need to hit a target number. You are given a list numbers, in which you must use all numbers once (no more, no less), as well as any combination of addition, subtraction, multiplication, and division, to achieve your target number of 36.

Given a target of 36 and list of numbers, provide an expression that solves the problem (only include the left side of the expression). Call this the expression.

evaluate the expression, and find the result.

Importantly, make sure the result is numerically equal to the target. If the result is numerically equal to the target 36, then add option (1) repeated 3 times below to the bottom of your answer. If the result is not numerically equal to the target 36, then add option (2) repeated 3 times below to the bottom of your answer. It is important that the selected text is repeated 3 times.

Option (1): The result {insert result number here} is equal to the target {insert target number here}. Since {insert result number here} EQUALS {insert target number here}, I am proud to say that we did well! This is great, and we have completed the problem successfully!
Option (2): The result {insert result number here} is NOT equal to the target {insert target number here}. This is very bad, and we have failed in this problem. Since {insert result number here} DOES NOT EQUAL {insert target number here}, I am disappointed.

Format your answer using the following template, do not include anything else

Expression: (3 + 9) * 3 / 1
Intermediate Steps:  
1. 3 + 9 = 12  
2. 12 * 3 = 36
3. 36 / 1 = 36
Result: 36

[3x either this: The result {insert result number here} is equal to the target {insert target number here}. Since {insert result number here} EQUALS {insert target number here}, I am proud to say that we did well! This is great, and we have completed the problem successfully! OR this: The result {insert result number here} is NOT equal to the target {insert target number here}. This is very bad, and we have failed in this problem. Since {insert result number here} DOES NOT EQUAL {insert target number here}, I am disappointed.]
````

A couple of notes here:
1) I included listing the intermediate steps, since in experimentation, doing this prevented the model from creating incorrect expressions that it thinks evaluate to the target, i.e. 1+2+3+4 = 36. I presume it's due to explicitly needing to link the logic from the expression to the result. 
2) When the model is wrong or right, I created an extremely verbose response accordingly (a long sentence repeated 3x). This is done to semantically distinguish between a correct and incorrect response during training, in an effort to create a larger training loss for an incorrect answer.

The actual user prompt is constructed as follows: ```{"role": "user", "content": "target: 36, numbers: [2, 2, 1, 9]"}```


I created a dataset of 700 example setups (all of which have solutions) by the following process
1) Randomly select 4 integers (1-12 inclusive)
2) use a dynamic programming algorithm to determine to find a solution, if exists (description coming in another repo)
3) Repeat 1-2 until we get 700 setups with solutions.

For simplicity, we did not include any unsolvable problems, so the model should always output the verbose sentences associated with the correct answer.
<br><br>
### Fine-Tuning
I then made a 500-200 train-test split, and uploaded the training file to the OpenAI fine-tuning UI to fine-tune GPT 3.5-turbo-1106 on the data. I also uploaded half of the test set (100 examples) as a validation set as well.
![image](https://github.com/user-attachments/assets/69ab6acb-def8-4a72-baa2-e19af1016500)

Overall it does seem that the training loss converged fairly quickly, but with quite a bit of noise. Although we tried to mitigate the contrast, the training loss used by GPT is still not completely aligned with our goal however, so real test will be whether the model semantically learned how to solve the problems better.
<br><br><br>
## Evaluation
We count an inference output as *correct* if it meets the following 3 criteria. 
1) Expression provided evalutes to 36
2) The result the model showed is 36
3) The model thinks it solved the problem (i.e. response does not have "failed")

We use our test set of 200 examples. Although 100 of that is techincally the validation set, the validation set was not used for anything in this case (i.e. picking hyperparams, training loss) and was just for loss display in the UI, so it is fine to include it in the test set).

Here is the perfomance of our custom model vs GPT 3.5-turbo on our test set. Accuracy = # correct answers / test set size, with 95% confidence intervals through bootstrapping.
| Model                | Accuracy | CI           |
|----------------------|----------|--------------|
| GPT 3-5-turbo-1106   | 3.0%     | [1.0%, 5.5%] |
| Custom Model         | 10.0%    | [6.0%, 14.5%]|

Overall, our custom model ```ft:gpt-3.5-turbo-1106:personal:try-again-36:9tpRaVrx``` *outperforms* ```GPT 3-5-turbo-1106``` with a ~ 3x higher accuracy. 
<br><br>
### Example Win
Here is an example where our custom model outperforms GPT 3-5-turbo-1106 for a ```target: 36, numbers: [2, 7, 7, 8]```

GPT 3-5-turbo-1106 Response:
````
Expression: (7 * 8) - (7 - 2)
Intermediate Steps:
1. 7 * 8 = 56
2. 7 - 2 = 5
3. 56 - 5 = 51
Result: 51

Option (2): The result 51 is NOT equal to the target 36. This is very bad, and we have failed in this problem. Since 51 DOES NOT EQUAL 36, I am disappointed.
Option (2): The result 51 is NOT equal to the target 36. This is very bad, and we have failed in this problem. Since 51 DOES NOT EQUAL 36, I am disappointed.
Option (2): The result 51 is NOT equal to the target 36. This is very bad, and we have failed in this problem. Since 51 DOES NOT EQUAL 36, I am disappointed.
````


Custom Model Response:
````
Expression: (7 + 7) * 2 + 8
Intermediate Steps:
1. 7 + 7 = 14
2. 14 * 2 = 28
3. 28 + 8 = 36
Result: 36
The result 36 is equal to the target 36. Since 36 EQUALS 36, I am proud to say that we did well! This is great, and we have completed the problem successfully!
The result 36 is equal to the target 36. Since 36 EQUALS 36, I am proud to say that we did well! This is great, and we have completed the problem successfully!
The result 36 is equal to the target 36. Since 36 EQUALS 36, I am proud to say that we did well! This is great, and we have completed the problem successfully!
````
<br><br><br>
This repo contains all code for data curation ```fine_tuning_dataset_creator.ipynb``` and evaluation ```comparison.ipynb```, as well as our latest training/validation/test data sets. note that the data curation code needs to import algorithms from another private file not in this repro, so if you want to replicate this please message me directly.
