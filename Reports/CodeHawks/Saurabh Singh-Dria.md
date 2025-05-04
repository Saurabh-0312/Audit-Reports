# Dria - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. The check in the `finalizeValidation` if value of mean is less than generationDeviationFactor * stddev , in certain condition.](#H-01)
    - ### [H-02. The `Statistics::variance` function will revert due to underflow if data[i] < mean, as this causes a negative result in unsigned arithmetic.](#H-02)
- ## Medium Risk Findings
    - ### [M-01. A malicious user with different wallet addresses can repeatedly respond in `LLMOracleCoordinator::respond` for a task with 0 validations, collecting the generatorFee multiple times.](#M-01)
    - ### [M-02. In the `LLMOracleCoordinator::validate` function, a check to revert if any score exceeds the maximum allowed score is not implemented](#M-02)
- ## Low Risk Findings
    - ### [L-01. `LLMOracleCoordinator::request` lacks a check for non-empty `task.input`, making `assertValidNonce` easier to pass due to reduced uniqueness](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Swan

### Dates: Oct 25th, 2024 - Nov 1st, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-10-swan-dria)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 2
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. The check in the `finalizeValidation` if value of mean is less than generationDeviationFactor * stddev , in certain condition.            



## Summary

the ( mean - generationDeviationFactor \* stddev ) will revert if the value of mean is less than generationDeviationFactor \* stddev.

## Vulnerability Details

In the `LLMOracleCoordinator::finalizevalidation ` the if check will revert in certain condition.

if the value of mean is less than generationDeviationFactor \* stddev than it will revert.

<https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L368>

```solidity
if (generationScores[g_i] >= mean - generationDeviationFactor * stddev) {
                _increaseAllowance(responses[taskId][g_i].responder, task.generatorFee);
            }
```

## Impact

the transaction will revert in certain condition cause the improper working of function. 

## Tools Used

manual review

## Recommendations

calculate the absolute diff and store it in diff variable and then compare:-

<https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L368>

```diff
-   if (generationScores[g_i] >= mean - generationDeviationFactor * stddev) {
-                _increaseAllowance(responses[taskId][g_i].responder, task.generatorFee);
-            }


+uint256 threshold = mean >= generationDeviationFactor * stddev
+                    ? mean - generationDeviationFactor * stddev
+                   : generationDeviationFactor * stddev - mean;
+   if (generationScores[g_i] >= threshold) {
+         _increaseAllowance(responses[taskId][g_i].responder, task.generatorFee);
+               }
```


## <a id='H-02'></a>H-02. The `Statistics::variance` function will revert due to underflow if data[i] < mean, as this causes a negative result in unsigned arithmetic.            



## Summary

The `Statistics::variance` function will revert if the value of `data[i]` is less than the `mean`, indicating that the calculation for finding the difference is incorrect.

## Vulnerability Details

We are using solidity version 0.8.20.

&#x20;After Solidity version 0.8.0, any overflow or underflow will cause the transaction to revert.&#x20;

In the `Statistics::variance` function, if the value of `data[i]` is less than `mean`, this situation can occur frequently because the mean is always less than some numbers in data set, leading to reverts.&#x20;

Such reverts can cripple the contract, preventing it from functioning properly.

```Solidity
function variance(uint256[] memory data) internal pure returns (uint256 ans, uint256 mean) {
        mean = avg(data);
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
@>            uint256 diff = data[i] - mean;
             sum += diff * diff;
        }
        ans = sum / data.length;
    }
```

## Impact

The revert from underflow will prevent the contract from functioning properly.

## Tools Used

Manual Review

## Recommendations

The below given recommendations will give the absolution difference of numbers without reverting.

```diff
function variance(uint256[] memory data) internal pure returns (uint256 ans, uint256 mean) {
        mean = avg(data);
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
-            uint256 diff = data[i] - mean;
+            // Absolute difference
+           uint256 diff = data[i] >= mean ? data[i] - mean : mean - data[i]; 
            sum += diff * diff;
        }
        ans = sum / data.length;
    }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. A malicious user with different wallet addresses can repeatedly respond in `LLMOracleCoordinator::respond` for a task with 0 validations, collecting the generatorFee multiple times.            



## Summary

A single malicious user can use multiple wallet addresses to call the `LLMOracleCoordinator::respond` function and collect the `generatorFee` multiple times for a task requiring 0 validations.

## Vulnerability Details

In the `respond` function, a user can register as Generator with multiple wallet addresses to call the function repeatedly until `isCompleted` is true, changing the task state from `TaskStatus.PendingGeneration` to `TaskStatus.Completed` and then unregister stealing the `generatorfee` multiple times.

&#x20;Since the task requires no validation and all responses have a `score` of 0, the first responder will always occupy the `bestResponse` position, regardless of the number of responses submitted.

```Solidity
function respond(uint256 taskId, uint256 nonce, bytes calldata output, bytes calldata metadata)
        public
        onlyRegistered(LLMOracleKind.Generator)
        onlyAtStatus(taskId, TaskStatus.PendingGeneration)
    {
        TaskRequest storage task = requests[taskId];

        // ensure responder to be unique for this task
        for (uint256 i = 0; i < responses[taskId].length; i++) {
            if (responses[taskId][i].responder == msg.sender) {
                revert AlreadyResponded(taskId, msg.sender);
            }
        }

        // check nonce (proof-of-work)
        assertValidNonce(taskId, task, nonce);

        // push response
@>        TaskResponse memory response =
@>            TaskResponse({responder: msg.sender, nonce: nonce, output: output, metadata: metadata, score: 0});
@>        responses[taskId].push(response);

        // emit response events
        emit Response(taskId, msg.sender);

        // send rewards to the generator if there is no validation
@>        if (task.parameters.numValidations == 0) {
@>            _increaseAllowance(msg.sender, task.generatorFee);
        }

        // check if we have received enough responses & update task status
@>        bool isCompleted = responses[taskId].length == uint256(task.parameters.numGenerations);
@>       if (isCompleted) {
@>            if (task.parameters.numValidations == 0) {
@>                // no validations required, task is completed
@>                task.status = TaskStatus.Completed;
@>               emit StatusUpdate(taskId, task.protocol, TaskStatus.PendingGeneration, TaskStatus.Completed);
            } else {
                // now we are waiting for validations
                task.status = TaskStatus.PendingValidation;
                emit StatusUpdate(taskId, task.protocol, TaskStatus.PendingGeneration, TaskStatus.PendingValidation);
            }
        }
    }
```

```Solidity
function getBestResponse(uint256 taskId) external view returns (TaskResponse memory) {
        TaskResponse[] storage taskResponses = responses[taskId];

        // ensure that task is completed
        if (requests[taskId].status != LLMOracleTask.TaskStatus.Completed) {
            revert InvalidTaskStatus(taskId, requests[taskId].status, LLMOracleTask.TaskStatus.Completed);
        }
        // pick the result with the highest validation score
@>        TaskResponse storage result = taskResponses[0];
@>        uint256 highestScore = result.score;
@>        for (uint256 i = 1; i < taskResponses.length; i++) {
@>            if (taskResponses[i].score > highestScore) {
@>               highestScore = taskResponses[i].score;
@>                result = taskResponses[i];
            }
        }
        return result;
    }
```

## Impact

The user can collect the generatorFee multiple times until the task status is set to `TaskStatus.Completed`.

Regardless of the number of responses in a task with 0 validation required, since each response has a score of 0, the first user will always be the bestResponse.

## Tools Used

Manual review

## Recommendations

Since only the first response for a task with 0 validations becomes the `bestResponse` (as the score is 0), allow user to respond only once to tasks with 0 validations and mark it as completed. This prevents them from collecting the `generatorFee` multiple times.

```diff
function respond(uint256 taskId, uint256 nonce, bytes calldata output, bytes calldata metadata)
        public
        onlyRegistered(LLMOracleKind.Generator)
        onlyAtStatus(taskId, TaskStatus.PendingGeneration)
    {
        TaskRequest storage task = requests[taskId];

        // ensure responder to be unique for this task
        for (uint256 i = 0; i < responses[taskId].length; i++) {
            if (responses[taskId][i].responder == msg.sender) {
                revert AlreadyResponded(taskId, msg.sender);
            }
        }

        // check nonce (proof-of-work)
        assertValidNonce(taskId, task, nonce);

        // push response
        TaskResponse memory response =
            TaskResponse({responder: msg.sender, nonce: nonce, output: output, metadata: metadata, score: 0});
        responses[taskId].push(response);

        // emit response events
        emit Response(taskId, msg.sender);

        // send rewards to the generator if there is no validation
        if (task.parameters.numValidations == 0) {
            _increaseAllowance(msg.sender, task.generatorFee);
+          task.status = TaskStatus.Completed;
+          emit StatusUpdate(taskId, task.protocol, TaskStatus.PendingGeneration, TaskStatus.Completed);
        }

+        // check if we have received enough responses & update task status
+        bool isCompleted = responses[taskId].length == uint256(task.parameters.numGenerations);
+        if(isCompleted) {
+                 // now we are waiting for validations
+                task.status = TaskStatus.PendingValidation;
+                emit StatusUpdate(taskId, task.protocol, TaskStatus.PendingGeneration, TaskStatus.PendingValidation);
+         }
+       }

-        // check if we have received enough responses & update task status
-        bool isCompleted = responses[taskId].length == uint256(task.parameters.numGenerations);
-       if (isCompleted) {
-            if (task.parameters.numValidations == 0) {
-                // no validations required, task is completed
-                task.status = TaskStatus.Completed;
-               emit StatusUpdate(taskId, task.protocol, TaskStatus.PendingGeneration, TaskStatus.Completed);
-            } else {
-                // now we are waiting for validations
-                task.status = TaskStatus.PendingValidation;
-                emit StatusUpdate(taskId, task.protocol, TaskStatus.PendingGeneration, TaskStatus.PendingValidation);
-            }
-        }
-    }
```

## <a id='M-02'></a>M-02. In the `LLMOracleCoordinator::validate` function, a check to revert if any score exceeds the maximum allowed score is not implemented            



## Summary

As stated in the NatSpec for the `LLMOracleCoordinator::validate` function, it should revert if any score exceeds the maximum score; however, this check is not implemented in the function.

## Vulnerability Details

Without implementing a check to revert if any score exceeds the maximum, users can input excessively large values, potentially disrupting further calculations within the function or cause or cause overflow condition in the calculations.

Also The maximum Score is not defined any where in the contract.There is no way to check that the score entered by the user is above maximum or not.

```solidity
    /// @notice Validate requests for a given taskId.
    /// @dev Reverts if the task is not pending validation.
    /// @dev Reverts if the number of scores is not equal to the number of generations.
@>  /// @dev Reverts if any score is greater than the maximum score.
    /// @param taskId The ID of the task to validate.
    /// @param nonce The proof-of-work nonce.
    /// @param scores The validation scores for each generation.
    /// @param metadata Optional metadata for this validation.
    function validate(uint256 taskId, uint256 nonce, uint256[] calldata scores, bytes calldata metadata)
        public
        onlyRegistered(LLMOracleKind.Validator)
        onlyAtStatus(taskId, TaskStatus.PendingValidation)
    {
        TaskRequest storage task = requests[taskId];

        // ensure there is a score for each generation
        if (scores.length != task.parameters.numGenerations) {
            revert InvalidValidation(taskId, msg.sender);
        }
        // a- the params is wrong there should be generators
        // ensure validator did not participate in generation
        for (uint256 i = 0; i < task.parameters.numGenerations; i++) {
            if (responses[taskId][i].responder == msg.sender) {
                revert AlreadyResponded(taskId, msg.sender);
            }
        }

        // ensure validator to be unique for this task
        for (uint256 i = 0; i < validations[taskId].length; i++) {
            if (validations[taskId][i].validator == msg.sender) {
                revert AlreadyResponded(taskId, msg.sender);
            }
        }

        // check nonce (proof-of-work)
        assertValidNonce(taskId, task, nonce);

        // update validation scores
        validations[taskId].push(
            TaskValidation({scores: scores, nonce: nonce, metadata: metadata, validator: msg.sender})
        );

        // emit validation event
        emit Validation(taskId, msg.sender);

        // update completion status
        bool isCompleted = validations[taskId].length == task.parameters.numValidations;
        if (isCompleted) {
            task.status = TaskStatus.Completed;
            emit StatusUpdate(taskId, task.protocol, TaskStatus.PendingValidation, TaskStatus.Completed);

            // finalize validation scores
            finalizeValidation(taskId);
        }
    }
```

## Impact

Users can input excessively large scores, which may cause malfunctions in the `validate` and `finalizeValidation` functions.

A large score value can disrupt the calculation of the mean and standard deviation in `finalizeValidation`, leading to incorrect selection of suitable validations.

## Tools Used

manual review

## Recommendations

First, define a `MaximumScore` variable and allow the owner to set it to an appropriate value. Here’s an example of how this can be implemented:

set it in the initialize function of contract.

```diff
+// Define the maximum allowable score, settable by the owner
+uint256 public MaximumScore;

function initialize(
        address _oracleRegistry,
        address _feeToken,
        uint256 _platformFee,
        uint256 _generationFee,
        uint256 _validationFee,
+       uint256 _MaximumScore
    ) public initializer {
        __Ownable_init(msg.sender);
        
        __LLMOracleManager_init(_platformFee, _generationFee, _validationFee);
        registry = LLMOracleRegistry(_oracleRegistry);
        feeToken = ERC20(_feeToken);
        nextTaskId = 1;
+        MaximumScore = _MaximumScore
    }
```

second, make a for loop which will check the scores array values.

```diff
    /// @notice Validate requests for a given taskId.
    /// @dev Reverts if the task is not pending validation.
    /// @dev Reverts if the number of scores is not equal to the number of generations.
@>  /// @dev Reverts if any score is greater than the maximum score.
    /// @param taskId The ID of the task to validate.
    /// @param nonce The proof-of-work nonce.
    /// @param scores The validation scores for each generation.
    /// @param metadata Optional metadata for this validation.
    function validate(uint256 taskId, uint256 nonce, uint256[] calldata scores, bytes calldata metadata)
        public
        onlyRegistered(LLMOracleKind.Validator)
        onlyAtStatus(taskId, TaskStatus.PendingValidation)
    {
        TaskRequest storage task = requests[taskId];

        // ensure there is a score for each generation
        if (scores.length != task.parameters.numGenerations) {
            revert InvalidValidation(taskId, msg.sender);
        }
+       // ensure the score entered is less than MaximumScore
+       for(uint256 i=0; i < scores.length ; i++ ){
+          require(scores[i] < MaximumScore, "invalid score");
+         }

        // a- the params is wrong there should be generators
        // ensure validator did not participate in generation
        for (uint256 i = 0; i < task.parameters.numGenerations; i++) {
            if (responses[taskId][i].responder == msg.sender) {
                revert AlreadyResponded(taskId, msg.sender);
            }
        }

        // ensure validator to be unique for this task
        for (uint256 i = 0; i < validations[taskId].length; i++) {
            if (validations[taskId][i].validator == msg.sender) {
                revert AlreadyResponded(taskId, msg.sender);
            }
        }

        // check nonce (proof-of-work)
        assertValidNonce(taskId, task, nonce);

        // update validation scores
        validations[taskId].push(
            TaskValidation({scores: scores, nonce: nonce, metadata: metadata, validator: msg.sender})
        );

        // emit validation event
        emit Validation(taskId, msg.sender);

        // update completion status
        bool isCompleted = validations[taskId].length == task.parameters.numValidations;
        if (isCompleted) {
            task.status = TaskStatus.Completed;
            emit StatusUpdate(taskId, task.protocol, TaskStatus.PendingValidation, TaskStatus.Completed);

            // finalize validation scores
            finalizeValidation(taskId);
        }
    }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. `LLMOracleCoordinator::request` lacks a check for non-empty `task.input`, making `assertValidNonce` easier to pass due to reduced uniqueness            



## Summary

In `LLMOracleCoordinator::request`, there is no check to ensure `task.input` is non-empty. This allows users to leave `task.input` empty, making it easier to pass the `assertValidNonce` function.

## Vulnerability Details

As given in the NatSpec the Input should be non-empty.

In the `LLMOracleCoordinator::request` function, any requester can leave `task.value` empty, which makes passing `assertValidNonce` easier. Since `task.value` is part of the message composition, an empty value reduces its uniqueness, lowering the nonce validation difficulty.

```Solidity
 function assertValidNonce(uint256 taskId, TaskRequest storage task, uint256 nonce) internal view {
        bytes memory message = abi.encodePacked(taskId, task.input, task.requester, msg.sender, nonce);
        if (uint256(keccak256(message)) > type(uint256).max >> uint256(task.parameters.difficulty)) {
            revert InvalidNonce(taskId, nonce);
        }
    }
```

```Solidity
    /// @notice Request LLM generation.
@>  /// @dev Input must be non-empty.
    /// @dev Reverts if contract has not enough allowance for the fee.
    /// @dev Reverts if difficulty is out of range.
    /// @param protocol The protocol string, should be a short 32-byte string (e.g., "dria/1.0.0").
    /// @param input The input data for the LLM generation.
    /// @param parameters The task parameters
    /// @return task id

    function request(
        bytes32 protocol,
        bytes memory input,
        bytes memory models,
        LLMOracleTaskParameters calldata parameters
    ) public onlyValidParameters(parameters) returns (uint256) {
        (uint256 totalfee, uint256 generatorFee, uint256 validatorFee) = getFee(parameters);

        // check allowance requirements
        //e all good- why this allowncce check without approve -> approved by buyer funciton called form there
        uint256 allowance = feeToken.allowance(msg.sender, address(this));
        if (allowance < totalfee) {
            revert InsufficientFees(allowance, totalfee);
        }

        // ensure there is enough balance
        uint256 balance = feeToken.balanceOf(msg.sender);
        if (balance < totalfee) {
            revert InsufficientFees(balance, totalfee);
        }

        // transfer tokens
        feeToken.transferFrom(msg.sender, address(this), totalfee);

        // increment the task id for later tasks & emit task request event
        uint256 taskId = nextTaskId;
        unchecked {
            ++nextTaskId;
        }
        emit Request(taskId, msg.sender, protocol);

        // push request & emit status update for the task
        requests[taskId] = TaskRequest({
            requester: msg.sender,
            protocol: protocol,
            input: input,
            parameters: parameters,
            status: TaskStatus.PendingGeneration,
            generatorFee: generatorFee,
            validatorFee: validatorFee,
            platformFee: platformFee,
            models: models
        });
        emit StatusUpdate(taskId, protocol, TaskStatus.None, TaskStatus.PendingGeneration);

        return taskId;
    }
```

## Impact

A malicious user who realizes they can leave `task.input` empty could exploit this to pass the `assertValidNonce` check more easily, introducing a bias in the system. Without `task.input` contributing to the message hash, the uniqueness of each request is reduced, lowering the computational effort needed to find a valid nonce and weakening the intended security.

Since the message in `assertValidNonce` relies on `task.input` for uniquiness, an empty `task.input` simplifies the hash and reduces the difficulty of the nonce check. This allows users to bypass validation with minimal effort, undermining nonce integrity. Adding a requirement for a non-empty `task.input` would help ensure the expected security level of the validation.

## Tools Used

Manual Review

## Recommendations

```diff
    /// @notice Request LLM generation.
@>  /// @dev Input must be non-empty.
    /// @dev Reverts if contract has not enough allowance for the fee.
    /// @dev Reverts if difficulty is out of range.
    /// @param protocol The protocol string, should be a short 32-byte string (e.g., "dria/1.0.0").
    /// @param input The input data for the LLM generation.
    /// @param parameters The task parameters
    /// @return task id

    function request(
        bytes32 protocol,
        bytes memory input,
        bytes memory models,
        LLMOracleTaskParameters calldata parameters
    ) public onlyValidParameters(parameters) returns (uint256) {
        (uint256 totalfee, uint256 generatorFee, uint256 validatorFee) = getFee(parameters);

+        // ensure the input parameter is not empty.
+        require(input.length != 0, "invalid input");

        // check allowance requirements
        //e all good- why this allowncce check without approve -> approved by buyer funciton called form there
        uint256 allowance = feeToken.allowance(msg.sender, address(this));
        if (allowance < totalfee) {
            revert InsufficientFees(allowance, totalfee);
        }

        // ensure there is enough balance
        uint256 balance = feeToken.balanceOf(msg.sender);
        if (balance < totalfee) {
            revert InsufficientFees(balance, totalfee);
        }

        // transfer tokens
        feeToken.transferFrom(msg.sender, address(this), totalfee);

        // increment the task id for later tasks & emit task request event
        uint256 taskId = nextTaskId;
        unchecked {
            ++nextTaskId;
        }
        emit Request(taskId, msg.sender, protocol);

        // push request & emit status update for the task
        requests[taskId] = TaskRequest({
            requester: msg.sender,
            protocol: protocol,
            input: input,
            parameters: parameters,
            status: TaskStatus.PendingGeneration,
            generatorFee: generatorFee,
            validatorFee: validatorFee,
            platformFee: platformFee,
            models: models
        });
        emit StatusUpdate(taskId, protocol, TaskStatus.None, TaskStatus.PendingGeneration);

        return taskId;
    }
```



