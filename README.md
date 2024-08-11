# Fine Tuned GPT 3.5 for 24-style Math Puzzle
I fine-tuned GPT 3.5-turbo to be better at a math puzzle game, specifically a 24-style game. In this game, you are given 4 numbers, and you must add/subtract/multiply/divide between them to reach a target number. You must use all the numbers.

GPT is notoriously bad at precise math calculations, so I wanted to see if fine-tuning could improve it's performance. I used a target number of 36 instead of 24 since it had a larger problem space.



## Training
I generated hundreds of different examples of 4 numbers being used to hit 36. I constructed each training example for the fine-tuning job as follows:

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
1) I included listing the intermediate steps, since in experimentation, doing this prevented the model from creating incorrect expressions that lead to the target, i.e. 1+2+3+4 = 36. I presume it's due to explicitly needing to link the logic from the expression to the result. 
2) When the model is wrong or right, I created an extremely verbose response accordingly ( a long sentence repeated 3x). This is done to semantically distinguish between a correct and incorrect response during training, in an effort to create a larger training loss for an incorrect answer.

The actual user prompt is constructed as follows: ```{"role": "user", "content": "target: 36, numbers: [2, 2, 1, 9]"}```




