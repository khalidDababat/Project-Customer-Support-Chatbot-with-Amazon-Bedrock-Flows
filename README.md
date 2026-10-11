# Amazon Bedrock AgentCore Customer Support Chatbot

## Project Overview

This project implements an AI-powered customer support chatbot using **Amazon Bedrock AgentCore Managed Harness** and **AgentCore Gateway**.

The chatbot supports three mutually exclusive request categories:

1. **Bug Reports:** Collects the issue description, reproduction steps, and environment details before creating a ticket using an AWS Lambda tool.
2. **Platform Questions:** Answers customer questions using the provided online shop FAQ without inventing unsupported information.
3. **Other Requests:** Politely redirects requests outside the chatbot's support scope to the support phone line.

Bug reports are stored in Amazon DynamoDB through the `bugreports___create_bug_report` tool.

## Architecture

- **AI Model:** Amazon Nova Pro
- **Agent Framework:** Amazon Bedrock AgentCore Managed Harness
- **Tool Integration:** Amazon Bedrock AgentCore Gateway
- **Backend Tool:** AWS Lambda
- **Database:** Amazon DynamoDB
- **Evaluation:** Amazon Bedrock Evaluations (LLM-as-a-judge)
- **Evaluation Metric:** `Builtin.Correctness`
- **AWS Region:** `us-east-1`

The Managed Harness handles customer interactions and routes requests according to the instructions defined in `system_prompt.txt`.

For bug reports, the chatbot invokes the Lambda tool through AgentCore Gateway after collecting all required information. The Lambda function stores the ticket in DynamoDB.

For platform questions, the chatbot uses the official FAQ as its authoritative source.

## Request Routing

### 1. Bug Report

When a customer reports a technical issue, the chatbot collects three required fields:

- `description`: A clear explanation of the issue.
- `stepsToReproduce`: The actions required to reproduce the issue.
- `environment`: The browser, operating system, device, or other relevant environment information.

If any field is missing, the chatbot should ask a follow-up question rather than invoke the tool.

Once all three fields are available, it invokes:

`bugreports___create_bug_report`

The tool creates a ticket in the DynamoDB table:

`bug-report-tool-stack-bug-reports`

After successful creation, the chatbot provides the customer with the returned ticket ID.

### 2. Platform Questions

The chatbot answers questions using only the information provided in `online_shop_faq.md`.

For questions covered by the FAQ, it provides concise, accurate answers.

For questions not covered by the FAQ, it acknowledges the missing information and directs the customer to the support phone line.

The chatbot must not invent unsupported policies, shipping destinations, prices, or other platform-specific details.

### 3. Other Requests

For requests unrelated to online shopping platform support, the chatbot politely explains its scope and directs the customer to the support phone line.

It does not fulfill unrelated requests or invoke the bug-report tool.

## Project Files

| File | Description |
|---|---|
| `system_prompt.txt` | System instructions for routing, bug-report collection, FAQ handling, and support handoffs |
| `agentcore_config.json` | Generated AgentCore configuration, including Harness and Gateway ARNs |
| `chat.py` | Interactive chatbot testing script |
| `create_harness.py` | Creates or configures the AgentCore Managed Harness |
| `setup_gateway.py` | Configures the AgentCore Gateway |
| `harness-tests.json` | Automated test cases covering all three routing categories |
| `generate-eval-dataset.py` | Generates the evaluation dataset |
| `output_eval_dataset.jsonl` | Generated evaluation records |
| `online_shop_faq.md` | Official FAQ knowledge source |
| `bug-report-transcript.txt` | Successful multi-turn bug-report conversation |
| `faq-covered-transcript.txt` | Covered FAQ question transcript |
| `faq-uncovered-transcript.txt` | Uncovered FAQ question transcript |
| `other-request-transcript.txt` | Out-of-scope request transcript |

Screenshots of DynamoDB and the completed Bedrock Evaluation job are included as separate submission evidence.

## Automated Testing

Automated testing was performed using `generate-eval-dataset.py` and four test cases defined in `harness-tests.json`.

The test cases covered:

1. Bug Report
2. FAQ Covered Question
3. FAQ Uncovered Question
4. Other Request

All four Harness calls completed successfully.

The generated `output_eval_dataset.jsonl` contained four evaluation records without any `[HARNESS_ERROR]` entries.

The evaluation dataset was uploaded to Amazon S3 and evaluated using Amazon Bedrock Evaluations.

The evaluation used:

- **Evaluation Job:** `support-chatbot-eval-run-1`
- **Evaluation Metric:** `Builtin.Correctness`
- **Judge Model:** Amazon Nova Pro

## Evaluation Results

| Test Case | Correctness Score |
|---|---|
| Bug Report | 0.5 |
| FAQ Covered Question | 1.0 |
| FAQ Uncovered Question | 1.0 |
| Other Request | 1.0 |
| **Average Correctness** | **0.875 (87.5%)** |

The evaluation job completed successfully.

Three test cases received perfect correctness scores. The Bug Report test received a score of 0.5.

These scores represent the original automated evaluation. Subsequent manual tests were performed after system-prompt refinements and are described separately below.

## Observations and Analysis

### 1. Bug Report — Score: 0.5

In the automated evaluation, the chatbot correctly identified the technical issue and asked the customer for steps to reproduce the problem.

It did not indicate that a ticket had been created before all required information was collected.

However, the LLM-as-a-judge assigned a correctness score of 0.5, indicating that the response only partially satisfied the evaluator's expectations.

During additional manual testing, some conversations resulted in premature tool invocation or duplicate ticket behavior.

A later successful multi-turn conversation demonstrated the intended workflow:

1. The customer reported an error when uploading a profile picture.
2. The chatbot requested the steps to reproduce the issue.
3. The customer provided the reproduction steps.
4. The chatbot requested environment information.
5. The customer specified Google Chrome on Windows 11.
6. The chatbot invoked `bugreports___create_bug_report`.
7. The chatbot returned a ticket ID after successful creation.

**Verified Ticket ID:**

`8410c52a-162c-4912-b282-9232d310ab8f`

The corresponding record was verified in DynamoDB with the following fields:

- `ticketId`
- `description`
- `stepsToReproduce`
- `environment`
- `status`

The successful conversation is documented in `bug-report-transcript.txt`, with a matching DynamoDB screenshot.

**Proposed improvements:**

- Improve consistency in multi-turn information collection.
- Prevent premature tool invocation when required fields are missing.
- Reduce duplicate ticket creation.
- Expand automated testing to cover complete multi-turn bug-report workflows.

### 2. FAQ Covered Question — Score: 1.0

The chatbot correctly answered:

**Customer question:**

"Do I need an account to place an order?"

**Chatbot response:**

"No. You can check out as a guest. Creating an account lets you track orders, save addresses, and speed up future checkouts."

The response matches the official FAQ.

**Observation:**

The chatbot correctly retrieved the supported information without introducing unsupported claims.

**Proposed improvement:**

Expand test coverage to additional FAQ topics, including returns, refunds, shipping, and payment methods.

### 3. FAQ Uncovered Question — Score: 1.0

The chatbot was tested with the following question:

**Customer question:**

"Do you ship to Antarctica?"

The official FAQ does not explicitly confirm whether shipping to Antarctica is available.

The chatbot correctly avoided inventing a shipping policy.

Following the system-prompt refinement, a manual test produced this response:

**Chatbot response:**

"I can't confirm whether shipping to Antarctica is available. Please contact our support phone line for further assistance."

**Observation:**

The chatbot acknowledged the limitation of the available FAQ information and explicitly directed the customer to the support phone line, as requested in the reviewer feedback.

**Proposed improvement:**

Add more tests involving unsupported shipping destinations, delivery costs, and unavailable platform-policy information.

### 4. Other Request — Score: 1.0

The chatbot was tested with an unrelated programming request.

**Customer question:**

"Can you write a Python program to calculate a factorial?"

Following the system-prompt refinement, a manual test produced this response:

**Chatbot response:**

"I'm here to help with questions and technical issues related to our online shopping platform. For other assistance, please contact our support phone line."

**Observation:**

The chatbot correctly declined the unrelated request and directed the customer to the support phone line without generating programming code.

**Proposed improvement:**

Test additional out-of-scope requests to verify consistent routing across different topics.

## Additional Findings

### Multi-Turn Bug Report Reliability

Manual testing revealed that bug-report behavior was not always consistent.

Some conversations resulted in premature ticket submission or reuse of previously generated ticket information.

However, a subsequent successful test demonstrated that the chatbot could collect the required information across multiple turns before invoking the tool.

The successful bug-report transcript includes the required tool invocation:

`[tool call] bugreports___create_bug_report`

The returned ticket ID matches the record shown in the DynamoDB screenshot.

### Support Phone Line Handoff

The original FAQ provides the following verified contact options:

- The website help/contact form.
- Replying to an existing order email.

It does not specify a support telephone number.

Following the reviewer feedback, the system prompt was refined so that uncovered FAQ questions and out-of-scope requests explicitly refer customers to the **support phone line**.

Both routes were successfully verified through manual testing.

No telephone number was invented or presented as verified.

A real, verified support phone number would be needed before customers could use that handoff in a production environment.

### Evaluation Limitations

The automated evaluation included four test cases and reported an average correctness score of 87.5%.

The later successful manual tests provide additional evidence of the chatbot's behavior but do not replace the original evaluation results.

Broader testing would be necessary to measure reliability across a larger number of conversations and different customer inputs.

## Submission Evidence

The submission includes the following evidence:

1. Completed `system_prompt.txt`.
2. `agentcore_config.json` containing the AgentCore Harness and Gateway ARNs.
3. `bug-report-transcript.txt` showing the follow-up questions and the `[tool call] bugreports___create_bug_report` line.
4. A DynamoDB screenshot showing the bug-report record with the matching ticket ID.
5. `faq-covered-transcript.txt`.
6. `faq-uncovered-transcript.txt`.
7. `other-request-transcript.txt`.
8. `harness-tests.json`.
9. `output_eval_dataset.jsonl`.
10. This README containing evaluation results and observations.
11. A screenshot of the completed `support-chatbot-eval-run-1` evaluation results showing Correctness scores.

## Conclusion

The Amazon Bedrock AgentCore Customer Support Chatbot achieved an average correctness score of **0.875 (87.5%)** across four automated evaluation records.

Three test cases received perfect scores, while the Bug Report test received 0.5.

Additional manual testing successfully demonstrated all three routing categories, including:

- A complete multi-turn bug report with a verified DynamoDB record.
- Correct answers to questions covered by the FAQ.
- Appropriate support phone line handoffs for uncovered FAQ questions.
- Appropriate support phone line handoffs for out-of-scope requests.

The project demonstrates the integration of Amazon Bedrock AgentCore Managed Harness, AgentCore Gateway, AWS Lambda, and DynamoDB to implement an AI-powered customer support workflow.

Future improvements should focus on improving multi-turn bug-report reliability, preventing duplicate ticket creation, expanding evaluation coverage, and configuring a verified support phone number for production use.