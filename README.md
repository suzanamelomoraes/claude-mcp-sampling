# MCP Sampling and Progress Demo

This repository is part of the Model Context Protocol Advanced course and is used as a hands-on playground for learning Sampling.

## Setup

Ensure you've set the `ANTHROPIC_API_KEY` env variable with a valid key.

Install dependencies using uv:

```bash
uv sync
```

## Running the Project

Run the MCP client:

```bash
uv run client.py
```

---

# Sampling

Sampling is when an MCP server asks Claude to generate text on its behalf. It reverses the usual direction: instead of Claude calling the server, the server calls Claude. This shifts the responsibility and cost of text generation from the server to the client.

## Implementation

Setting up sampling requires code on both sides:

**Server Side**
In your tool function, use the create_message function to request text generation:

```typescript
@mcp.tool()
async def summarize(text_to_summarize: str, ctx: Context):
    prompt = f"""
    Please summarize the following text:
    {text_to_summarize}
    """

    result = await ctx.session.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=prompt
                )
            )
        ],
        max_tokens=4000,
        system_prompt="You are a helpful research assistant",
    )

    if result.content.type == "text":
        return result.content.text
    else:
        raise ValueError("Sampling failed")
```

**Client Side**
Create a sampling callback that handles the server's requests:

```typescript
async def sampling_callback(
    context: RequestContext, params: CreateMessageRequestParams
):
    # Call Claude using the Anthropic SDK
    text = await chat(params.messages)

    return CreateMessageResult(
        role="assistant",
        model=model,
        content=TextContent(type="text", text=text),
    )
```

Then pass this callback when initializing your client session:

```typescript
async with ClientSession(
    read,
    write,
    sampling_callback=sampling_callback
) as session:
    await session.initialize()
```

## When Should You Use It?

Use sampling when your MCP server needs AI reasoning embedded inside a workflow it's orchestrating — without forcing that logic back through the user or a separate API call you manage yourself.
Good fits:

- The server is running an autonomous multi-step task and needs a judgment call mid-way
- You want the server to evaluate, summarize, or classify something it just fetched before returning results
- You're building agentic pipelines where the server orchestrates subtasks that each need LLM reasoning

> **Rule of thumb:** if your server needs to think between steps, use sampling. If it just needs to act, use tools.
