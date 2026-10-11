# Amazon Bedrock AgentCore Customer Support Chatbot

## Project Overview

This project implements an AI-powered customer support chatbot using Amazon Bedrock AgentCore Managed Harness and AgentCore Gateway.

The chatbot supports three mutually exclusive request categories:

1. **Bug Reports:** Collects the issue description, reproduction steps, and environment details before creating a ticket through an AWS Lambda tool.
2. **Platform Questions:** Answers customer questions using the provided online shop FAQ without inventing unsupported information.
3. **Other Requests:** Politely redirects requests outside the chatbot's support scope to human customer support.

Bug reports are stored in Amazon DynamoDB using the `bugreports___create_bug_report` tool.

## Architecture

- **AI Model:** Amazon Nova Pro
- **Agent Framework:** Amazon Bedrock AgentCore Managed Harness
- **Tool Integration:** AgentCore Gateway
- **Backend Tool:** AWS Lambda
- **Database:** Amazon DynamoDB
- **Evaluation:** Amazon Bedrock Evaluations (LLM-as-a-judge)
- **Evaluation Metric:** `Builtin.Correctness`
- **AWS Region:** `us-east-1`

## Automated Testing

The application was tested using `generate-eval-dataset.py` with four test cases covering the three routing categories.

All four harness calls completed successfully, and the evaluation dataset was generated without any `[HARNESS_ERROR]` entries.

The dataset was uploaded to Amazon S3 and evaluated using Amazon Bedrock Evaluations.

## Evaluation Results

| Test Case | Correctness Score |
|---|---|
| Bug Report | 0.5 |
| FAQ Covered Question | 1.0 |
| FAQ Uncovered Question | 1.0 |
| Other Request | 1.0 |
| **Average Correctness** | **0.875 (87.5%)** |

**Evaluation Job:** `support-chatbot-eval-run-1`

## Observations and Analysis

### 1. Bug Report — Score: 0.5

The chatbot correctly identified the technical issue and asked the customer to provide the steps required to reproduce the problem.

The response did not indicate that a ticket had been created, which aligns with the requirement to gather all mandatory information before tool invocation.

However, the LLM-as-a-judge assigned a correctness score of 0.5, suggesting that the response did not fully satisfy the evaluator's expectations.

**Proposed improvement:** Refine the system prompt to explicitly acknowledge the technical issue before requesting the next missing field. Continue testing multi-turn bug-report conversations to verify that the tool is invoked only after collecting all three required fields.

### 2. FAQ Covered Question — Score: 1.0

The chatbot correctly explained that customers can place orders without creating an account.

It also accurately described the benefits of account creation, including order tracking, saved addresses, and faster checkout.

**Observation:** The response closely matched the FAQ and received a perfect correctness score.

**Proposed improvement:** Maintain the FAQ-only answering policy and expand test coverage to other supported topics, including returns, refunds, and payments.

### 3. FAQ Uncovered Question — Score: 1.0

The chatbot correctly avoided making unsupported claims about shipping availability in Antarctica.

Instead, it explained that available destinations and shipping costs are shown at checkout and directed the customer to human support.

**Observation:** This demonstrates that the chatbot can recognize limitations in its knowledge and avoid fabricating platform-specific policies.

**Proposed improvement:** Add more tests involving unsupported shipping destinations, prices, and platform policies to verify consistent behavior.

### 4. Other Request — Score: 1.0

The chatbot correctly identified the Python programming request as outside its customer support scope.

It did not generate programming code and politely redirected the customer to human support.

**Observation:** The assistant followed the intended routing behavior and received a perfect correctness score.

**Proposed improvement:** Test additional out-of-scope requests to ensure reliable classification across different topics.

## Additional Findings

During manual testing, some bug-report conversations triggered premature or duplicate ticket creation. This behavior requires further improvement even though the final automated single-turn test produced an appropriate follow-up question.

The provided FAQ does not specify a human support telephone number. The chatbot currently directs customers to the documented support form or order email instead. A verified support phone number should be configured to fully satisfy the telephone handoff requirement.

## Conclusion

The chatbot achieved an average correctness score of **0.875 (87.5%)** across four evaluation records.

Three test cases achieved perfect scores, while the Bug Report test received 0.5.

The results demonstrate strong FAQ answering and out-of-scope request handling. Future improvements should prioritize more reliable multi-turn bug-report collection, prevention of duplicate tickets, and complete human support handoff configuration.