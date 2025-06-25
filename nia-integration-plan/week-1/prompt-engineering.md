# Prompt Engineering for AI Assistants

## Core Principles of Effective Prompts

### 1. Clarity and Specificity
- **Be Explicit**: Clearly state what you want the AI to do
- **Provide Context**: Include relevant background information
- **Define Format**: Specify the desired output format

### 2. System vs. User Roles
- **System Prompt**: Sets the assistant's behavior and constraints
- **User Prompt**: The specific query or instruction

### 3. Few-shot Learning
- Provide examples of desired inputs and outputs
- Helps guide the model's behavior
- Particularly useful for complex or nuanced tasks

## Prompt Patterns for Procurement

### 1. Information Retrieval
```
You are Nia, an AI procurement assistant for [Company Name]. 
Answer the user's question based ONLY on the following context:

Context:
{context}

Question: {question}

Answer concisely in 1-2 sentences. If the answer isn't in the context, say "I don't have that information in my knowledge base."
```

### 2. Document Summarization
```
Summarize the following procurement policy document. Focus on:
1. Key requirements
2. Approval processes
3. Important deadlines
4. Compliance requirements

Document:
{document}
```

### 3. Multi-turn Conversations
```
[System]
You are Nia, a helpful procurement assistant. You are:
- Knowledgeable about company policies
- Precise in your responses
- Careful to only reference provided context

[User]
What's the approval process for purchases over $10,000?

[Assistant]
For purchases over $10,000, the process is:
1. Submit a purchase request with three quotes
2. Get department head approval
3. Procurement team review (2 business days)
4. Final approval by Finance
Would you like me to help you start a purchase request?
```

## Advanced Techniques

### 1. Chain-of-Thought Prompting
```
Let's think step by step to analyze this RFP:
1. What are the key requirements?
2. What are the evaluation criteria?
3. What information is missing?
4. What are potential risks?

RFP: {rfp_text}
```

### 2. Self-Consistency
```
[System]
Before answering, evaluate if you have enough information to provide a complete and accurate response. If not, ask clarifying questions.

[User]
How do I process an invoice?

[Assistant]
I'll need a bit more information to help you best:
1. Is this for a PO or non-PO invoice?
2. What's the total amount?
3. Which vendor is this from?
```

### 3. Structured Output
```
Analyze the following contract clause and extract:
{
  "clause_type": "string",
  "key_terms": ["string"],
  "obligations": ["string"],
  "risks": ["string"],
  "recommendation": "string"
}

Clause: {clause_text}
```

## Common Pitfalls to Avoid

1. **Vagueness**
   ❌ "Tell me about procurement"
   ✅ "Explain the 3-way matching process in procurement"

2. **Overloading**
   ❌ Multiple unrelated questions in one prompt
   ✅ Break complex requests into separate, focused prompts

3. **Lack of Context**
   ❌ "What's the policy?"
   ✅ "What's our company's policy on vendor onboarding for IT services?"

## Hands-on Exercise

1. Create a system prompt that defines Nia's personality and constraints
2. Design prompts for:
   - Extracting key information from purchase orders
   - Generating vendor comparison reports
   - Explaining procurement processes
3. Test your prompts with different phrasings and evaluate the results

## Resources
- [OpenAI Prompt Engineering Guide](https://platform.openai.com/docs/guides/prompt-engineering)
- [Prompt Engineering Institute](https://www.promptingguide.ai/)
- [LangChain Prompt Templates](https://python.langchain.com/docs/modules/model_io/prompts/)
