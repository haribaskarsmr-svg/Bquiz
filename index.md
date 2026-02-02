# Copilot Council Implementation Guide

## Quick Start: Using the Existing Tool

### Prerequisites
1. **GitHub Copilot Subscription** (Individual, Business, or Enterprise)
2. **GitHub Copilot CLI** installed

### Installation Steps

#### Step 1: Install GitHub Copilot CLI
```bash
# Install via Homebrew (macOS/Linux)
brew install github/gh-cli/copilot-cli

# Or download from GitHub releases
# https://github.com/github/copilot-cli/releases
```

#### Step 2: Install Copilot Council

**Option A: Homebrew (Recommended)**
```bash
brew tap openjny/tap
brew install copilot-council
```

**Option B: Download Binary**
```bash
# Linux
wget https://github.com/openjny/copilot-council/releases/latest/download/copilot-council_linux_amd64.tar.gz
tar -xzf copilot-council_linux_amd64.tar.gz
sudo mv copilot-council /usr/local/bin/

# macOS
wget https://github.com/openjny/copilot-council/releases/latest/download/copilot-council_darwin_amd64.tar.gz
tar -xzf copilot-council_darwin_amd64.tar.gz
sudo mv copilot-council /usr/local/bin/
```

**Option C: Build from Source**
```bash
git clone https://github.com/openjny/copilot-council.git
cd copilot-council
go build -o copilot-council ./cmd/copilot-council
sudo mv copilot-council /usr/local/bin/
```

#### Step 3: Verify Installation
```bash
copilot-council --version
```

### Basic Usage Examples

#### Example 1: Simple Question
```bash
copilot-council "What is the best approach to implement caching in a microservices architecture?"
```

#### Example 2: Custom Models
```bash
copilot-council --models claude-sonnet-4.5,gpt-5.2,gemini-3-pro-preview \
  "How should I structure a React application with TypeScript?"
```

#### Example 3: Different Aggregator
```bash
copilot-council --aggregator claude-sonnet-4.5 \
  "What are the trade-offs between REST and GraphQL?"
```

#### Example 4: Verbose Mode (See All Responses)
```bash
copilot-council --verbose \
  "What's the best way to handle authentication in a Next.js app?"
```

#### Example 5: Longer Timeout
```bash
copilot-council --timeout 120 \
  "Design a scalable event-driven architecture for an e-commerce platform"
```

---

## Building Your Own Council Pattern Implementation

If you want to build your own version or understand how it works, here's a complete implementation guide.

### Architecture Overview

```
┌─────────────┐
│   User      │
│  Question   │
└──────┬──────┘
       │
       v
┌─────────────────────────────────────────┐
│         STAGE 1: Query Models           │
│  ┌───────────┐  ┌───────────┐  ┌──────┐│
│  │ Claude    │  │   GPT     │  │Gemini││
│  │ Sonnet 4.5│  │   5.2     │  │ 3Pro ││
│  └─────┬─────┘  └─────┬─────┘  └───┬──┘│
└────────┼──────────────┼────────────┼───┘
         │              │            │
         v              v            v
    Response A     Response B    Response C
         │              │            │
         └──────┬───────┴────────────┘
                │
                v
┌───────────────────────────────────────────┐
│        STAGE 2: Peer Review               │
│  Each model reviews others' responses     │
│  (anonymized to prevent bias)             │
│                                           │
│  Claude reviews B & C                     │
│  GPT reviews A & C                        │
│  Gemini reviews A & B                     │
│                                           │
│  Output: Rankings + Reasoning             │
└──────────────────┬────────────────────────┘
                   │
                   v
┌───────────────────────────────────────────┐
│        STAGE 3: Synthesis                 │
│                                           │
│  Chairman Model (GPT-4.1) receives:       │
│  - All original responses (A, B, C)       │
│  - All peer reviews + rankings            │
│  - Original question                      │
│                                           │
│  Synthesizes the BEST answer              │
└──────────────────┬────────────────────────┘
                   │
                   v
              Final Answer
```

### Implementation in Go (Like the Original)

```go
package main

import (
    "context"
    "fmt"
    "sync"
)

// Stage 1: Query all models in parallel
func queryModels(ctx context.Context, question string, models []string) (map[string]string, error) {
    responses := make(map[string]string)
    var mu sync.Mutex
    var wg sync.WaitGroup
    errChan := make(chan error, len(models))

    for _, model := range models {
        wg.Add(1)
        go func(m string) {
            defer wg.Done()
            
            // Call Copilot API with the model
            response, err := callCopilotAPI(ctx, m, question)
            if err != nil {
                errChan <- fmt.Errorf("model %s failed: %w", m, err)
                return
            }
            
            mu.Lock()
            responses[m] = response
            mu.Unlock()
        }(model)
    }

    wg.Wait()
    close(errChan)
    
    // Check for errors
    if len(errChan) > 0 {
        return responses, <-errChan
    }
    
    return responses, nil
}

// Stage 2: Peer review - each model reviews others
func conductPeerReview(ctx context.Context, question string, responses map[string]string, models []string) (map[string][]Review, error) {
    reviews := make(map[string][]Review)
    var mu sync.Mutex
    var wg sync.WaitGroup

    for _, reviewer := range models {
        wg.Add(1)
        go func(r string) {
            defer wg.Done()
            
            // Get responses from other models (anonymized)
            otherResponses := make(map[string]string)
            responseLabels := make(map[string]string) // e.g., "Response A" -> actual model
            labelIdx := 'A'
            
            for model, response := range responses {
                if model != r {
                    label := fmt.Sprintf("Response %c", labelIdx)
                    otherResponses[label] = response
                    responseLabels[label] = model
                    labelIdx++
                }
            }
            
            // Ask reviewer to rank other responses
            reviewPrompt := buildReviewPrompt(question, otherResponses)
            reviewResult, err := callCopilotAPI(ctx, r, reviewPrompt)
            if err != nil {
                return
            }
            
            // Parse review and store
            parsedReview := parseReview(reviewResult, responseLabels)
            mu.Lock()
            reviews[r] = parsedReview
            mu.Unlock()
        }(reviewer)
    }

    wg.Wait()
    return reviews, nil
}

// Stage 3: Chairman synthesizes final answer
func synthesizeFinalAnswer(ctx context.Context, question string, responses map[string]string, 
                          reviews map[string][]Review, chairman string) (string, error) {
    
    synthesisPrompt := buildSynthesisPrompt(question, responses, reviews)
    finalAnswer, err := callCopilotAPI(ctx, chairman, synthesisPrompt)
    if err != nil {
        return "", err
    }
    
    return finalAnswer, nil
}

// Helper: Call Copilot API
func callCopilotAPI(ctx context.Context, model, prompt string) (string, error) {
    // This would use the Copilot SDK
    // For now, showing the structure
    cmd := exec.CommandContext(ctx, "copilot", "--model", model, prompt)
    output, err := cmd.CombinedOutput()
    if err != nil {
        return "", err
    }
    return string(output), nil
}

// Helper: Build review prompt
func buildReviewPrompt(question string, responses map[string]string) string {
    prompt := fmt.Sprintf(`You are reviewing other AI models' answers to this question:
"%s"

Please review and rank the following responses (anonymized):

`, question)
    
    for label, response := range responses {
        prompt += fmt.Sprintf("\n%s:\n%s\n", label, response)
    }
    
    prompt += `
Please provide:
1. Ranking (best to worst)
2. Reasoning for your rankings
3. Specific strengths and weaknesses of each

Format your response as:
RANKING: [A, B, C] (best to worst)
REASONING: [Your detailed analysis]
`
    
    return prompt
}

// Helper: Build synthesis prompt
func buildSynthesisPrompt(question string, responses map[string]string, reviews map[string][]Review) string {
    prompt := fmt.Sprintf(`You are the Chairman in a council of AI models. Your task is to synthesize 
the BEST possible answer based on multiple perspectives and peer reviews.

ORIGINAL QUESTION:
%s

RESPONSES FROM COUNCIL MEMBERS:
`, question)
    
    for model, response := range responses {
        prompt += fmt.Sprintf("\n%s:\n%s\n", model, response)
    }
    
    prompt += "\nPEER REVIEWS:\n"
    for reviewer, reviewList := range reviews {
        prompt += fmt.Sprintf("\n%s's Review:\n", reviewer)
        for _, review := range reviewList {
            prompt += fmt.Sprintf("  Ranked %s: %s\n", review.Target, review.Reasoning)
        }
    }
    
    prompt += `
Your task:
1. Analyze all responses and reviews objectively
2. Identify the strongest arguments and insights
3. Synthesize a definitive, well-reasoned answer
4. Avoid vague "it depends" statements - be decisive
5. If there are trade-offs, clearly state which option is better and why

Provide the BEST answer to the original question.
`
    
    return prompt
}

type Review struct {
    Target    string
    Rank      int
    Reasoning string
}

func main() {
    ctx := context.Background()
    question := "What is the best database for a high-traffic web application?"
    
    models := []string{
        "claude-sonnet-4.5",
        "gpt-5.2",
        "gemini-3-pro-preview",
    }
    chairman := "gpt-4.1"
    
    fmt.Println("Stage 1: Querying models...")
    responses, err := queryModels(ctx, question, models)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Println("Stage 2: Conducting peer review...")
    reviews, err := conductPeerReview(ctx, question, responses, models)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Println("Stage 3: Synthesizing final answer...")
    finalAnswer, err := synthesizeFinalAnswer(ctx, question, responses, reviews, chairman)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("\n=== FINAL ANSWER ===\n%s\n", finalAnswer)
}
```

### Implementation in Python (Simpler Alternative)

```python
import asyncio
from typing import Dict, List
import subprocess

class CouncilMember:
    def __init__(self, model_name: str):
        self.model = model_name
    
    async def answer(self, question: str) -> str:
        """Stage 1: Answer the question"""
        cmd = ["copilot", "--model", self.model, question]
        process = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        stdout, _ = await process.communicate()
        return stdout.decode()
    
    async def review(self, question: str, responses: Dict[str, str]) -> Dict[str, int]:
        """Stage 2: Review other responses"""
        # Create anonymized responses
        labels = {}
        review_text = f"Question: {question}\n\nResponses to review:\n"
        
        label = 'A'
        for model, response in responses.items():
            if model != self.model:
                labels[chr(ord(label))] = model
                review_text += f"\nResponse {label}:\n{response}\n"
                label = chr(ord(label) + 1)
        
        review_prompt = f"""{review_text}

Please rank these responses from best to worst and explain why.
Format: RANKING: [A, B, ...] (best to worst)
"""
        
        review = await self.answer(review_prompt)
        return self._parse_ranking(review, labels)
    
    def _parse_ranking(self, review: str, labels: Dict[str, str]) -> Dict[str, int]:
        # Parse the ranking from the review
        # Returns dict of model -> rank (1 = best)
        rankings = {}
        # Implementation would parse the review text
        return rankings

class CopilotCouncil:
    def __init__(self, models: List[str], chairman: str):
        self.members = [CouncilMember(m) for m in models]
        self.chairman = CouncilMember(chairman)
    
    async def ask(self, question: str) -> str:
        print("🏛️  Copilot Council - Consulting Models\n")
        
        # Stage 1: Get all responses in parallel
        print("Stage 1: Querying models...")
        tasks = [member.answer(question) for member in self.members]
        responses_list = await asyncio.gather(*tasks)
        
        responses = {
            member.model: response 
            for member, response in zip(self.members, responses_list)
        }
        
        for model, response in responses.items():
            print(f"  ✓ {model}")
        
        # Stage 2: Peer review
        print("\nStage 2: Conducting peer review...")
        review_tasks = [
            member.review(question, responses) 
            for member in self.members
        ]
        reviews = await asyncio.gather(*review_tasks)
        
        # Stage 3: Chairman synthesis
        print("\nStage 3: Synthesizing final answer...")
        synthesis_prompt = self._build_synthesis_prompt(
            question, responses, reviews
        )
        final_answer = await self.chairman.answer(synthesis_prompt)
        
        return final_answer
    
    def _build_synthesis_prompt(self, question: str, 
                                responses: Dict[str, str],
                                reviews: List[Dict]) -> str:
        prompt = f"""You are the Chairman synthesizing the best answer.

QUESTION: {question}

RESPONSES:
"""
        for model, response in responses.items():
            prompt += f"\n{model}:\n{response}\n"
        
        prompt += "\n\nBased on all responses and reviews, provide the BEST answer."
        return prompt

# Usage
async def main():
    council = CopilotCouncil(
        models=[
            "claude-sonnet-4.5",
            "gpt-5.2",
            "gemini-3-pro-preview"
        ],
        chairman="gpt-4.1"
    )
    
    answer = await council.ask(
        "What's the best approach to implement authentication in a microservices architecture?"
    )
    
    print("\n" + "="*60)
    print("FINAL ANSWER")
    print("="*60)
    print(answer)

if __name__ == "__main__":
    asyncio.run(main())
```

### Implementation Using Direct API Calls

If you want to implement this without the CLI, you can use the GitHub Models API directly:

```python
import anthropic
import openai
from typing import List, Dict
import asyncio

class APICouncil:
    def __init__(self, api_keys: Dict[str, str]):
        self.anthropic_client = anthropic.Anthropic(api_key=api_keys['anthropic'])
        self.openai_client = openai.OpenAI(api_key=api_keys['openai'])
    
    async def query_claude(self, prompt: str, model: str = "claude-sonnet-4-20250514") -> str:
        response = await asyncio.to_thread(
            self.anthropic_client.messages.create,
            model=model,
            max_tokens=2000,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.content[0].text
    
    async def query_gpt(self, prompt: str, model: str = "gpt-4") -> str:
        response = await asyncio.to_thread(
            self.openai_client.chat.completions.create,
            model=model,
            messages=[{"role": "user", "content": prompt}]
        )
        return response.choices[0].message.content
    
    async def run_council(self, question: str) -> str:
        # Stage 1: Get responses
        tasks = [
            self.query_claude(question, "claude-sonnet-4-20250514"),
            self.query_gpt(question, "gpt-4"),
        ]
        responses = await asyncio.gather(*tasks)
        
        # Stage 2 & 3: Review and synthesize
        # (Implementation similar to above)
        
        return final_answer

# Usage
async def main():
    council = APICouncil({
        'anthropic': 'your-anthropic-key',
        'openai': 'your-openai-key',
    })
    
    answer = await council.run_council("Your question here")
    print(answer)

asyncio.run(main())
```

---

## Advanced Tips

### 1. Customizing the Council

```bash
# Use more models for complex questions
copilot-council --models claude-sonnet-4.5,claude-opus-4.5,gpt-5.2,gpt-4.1,gemini-3-pro-preview \
  "Complex architectural decision..."

# Use faster models for quick questions
copilot-council --models claude-haiku-4.5,gpt-5-mini \
  "What does this error mean?"
```

### 2. Best Practices

- **Use 3-5 models** for most questions (optimal balance)
- **Choose diverse models** (Claude, GPT, Gemini mix)
- **Match timeout to complexity** (60s for simple, 120s+ for complex)
- **Use verbose mode** when you want to see individual perspectives
- **Chairman model matters** - GPT-4.1 is good at synthesis

### 3. When to Use Council Pattern

**Best for:**
- Architectural decisions
- Complex trade-off analysis
- Controversial technical topics
- Problems with multiple valid approaches
- High-stakes decisions

**Overkill for:**
- Simple factual questions
- Syntax questions
- Quick debugging
- Well-established best practices

---

## Troubleshooting

### Issue: "copilot command not found"
```bash
# Install GitHub Copilot CLI first
brew install github/gh-cli/copilot-cli

# Or check installation
which copilot
```

### Issue: "No Copilot subscription"
- You need an active GitHub Copilot subscription
- Sign up at https://github.com/features/copilot

### Issue: Timeout errors
```bash
# Increase timeout
copilot-council --timeout 180 "your question"
```

### Issue: Model not available
```bash
# Check available models
copilot --help

# Use available models only
copilot-council --models claude-sonnet-4.5,gpt-4.1 "question"
```

---

## Resources

- **Original Copilot Council**: https://github.com/openjny/copilot-council
- **LLM Council Pattern**: https://github.com/karpathy/llm-council
- **GitHub Copilot CLI**: https://github.com/github/copilot-cli
- **Copilot SDK**: https://github.com/github/copilot-sdk
- **Copilot Docs**: https://docs.github.com/copilot

---

## Next Steps

1. Install the prerequisites (Copilot subscription + CLI)
2. Install copilot-council via Homebrew
3. Try a simple question to verify it works
4. Experiment with different model combinations
5. Use verbose mode to understand the review process
6. Build your own custom implementation if needed

Enjoy having a council of AI models at your command! 🏛️
