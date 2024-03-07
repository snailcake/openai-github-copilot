## Why

> [GitHub Copilot](https://docs.github.com/en/copilot/quickstart) is free to use for
> <strong>verified</strong> [students], [teachers], and [maintainers] of <strong>popular</strong> open source projects.
<details>

- Students and teachers: visit [Benefits for everyone at school](https://education.github.com/benefits)
- Maintainers: check https://github.com/github-copilot/free_signup  
  If you are not eligible, it indicates that your project needs more stars.
</details>

Even if you don't meet the eligibility criteria, the paid subscription
([_Individual_](https://docs.github.com/en/copilot/copilot-individual/about-github-copilot-individual))
is more affordable than [OpenAI's offering](https://chat.openai.com/#pricing).
It costs only $10 and still provides access to [GPT-4](https://openai.com/gpt-4).

> [!NOTE]
> If the above options are not suitable for you - consider trying completely
> free alternative: [openai-gemini](https://github.com/PublicAffairs/openai-gemini).

[students]: https://github.com/pricing#i-am-a-student-can-i-get-access-to-github-copilot-for-free
[teachers]: https://github.com/pricing#i-am-a-teacher-can-i-get-access-to-github-copilot-for-free
[maintainers]: https://github.com/pricing#i-work-on-open-source-projects-can-i-get-access-to-github-copilot-for-free

## What

GitHub Copilot chat is exclusively available through selected IDEs.

The `openai-github-copilot` project enables its use across a broader range of tools.
It provides a general OpenAI-compatible API that can be deployed at no cost.  
(However, a GitHub Copilot subscription is still required.)


## Serverless?

Although it runs in the cloud, it does not require server maintenance.
It can be easily deployed to Cloudflare for free
(with generous [limits](https://developers.cloudflare.com/workers/platform/limits/#worker-limits)
suitable for personal use).

> [!TIP]
> Running the proxy endpoint locally is also an option,
> though it's more appropriate for development use.


## How to start

First, you need to create an account on [Cloudflare](https://dash.cloudflare.com/sign-up).

The simplest way to deploy is by manually pasting content of [`src/worker.mjs`](../cf-playground/src/worker.mjs)
into https://workers.cloudflare.com/playground (see there `Deploy` button).

> [!CAUTION]
> This is a bit simplified version of the code, that does not implement token caching,
> and lacks a "/token" page.  
> To deploy full-featured version, please refer to the section below.


### Advanced instructions

> [!TIP]
> If your account is new and you haven't created any workers yet, some of the links mentioned below
> might be inaccessible. Therefore, it's recommended to create at least a "Hello World" Worker:
> [Deploy "Hello World" script](https://dash.cloudflare.com/?to=/:account/workers-and-pages/create/workers/new).  
> You can name it "copilot".

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](
https://deploy.workers.cloudflare.com/?url=https://github.com/PublicAffairs/openai-github-copilot)

After clicking this button, you'll be guided through the process of forking the repository first,
which is necessary for continuous integration (CI).

> [!WARNING]
> The initial deployment will fail, because it requires _KV namespace_, which does not exist yet.

Navigate to [Workers & Pages / KV](https://dash.cloudflare.com/?to=/:account/workers/kv/namespaces)
and create a namespace with _"copilot-KV"_ name.
Next, copy its _ID_, and modify the [`wrangler.toml`](wrangler.toml) file in your forked repository,
replacing the existing `id` with your copied value.

Once you push these changes, the CI will automatically deploy the repo to Cloudflare Workers,
this time without any issues.


### Manual deployment

Instead of using the "_button-deploy_", you can also manage the project locally using the
[Cloudflare CLI](https://developers.cloudflare.com/workers/wrangler/):
- Create a _KV namespace_: `wrangler kv:namespace create KV`  
  (and update `id` value in your [`wrangler.toml`](wrangler.toml) file).
- Serve locally: `wrangler dev`
- Deploy to Cloudlare Workers: `wrangler deploy`


## How to use

If you open your newly-deployed site in a browser, you will only see a `404 Not Found` message.
This is expected, as the API is not designed for direct browser access.  
To utilize it, you should enter your _API address_ and your Github Copilot _access token_
into the corresponding fields in your software settings.

> [!NOTE]
> Not all software tools allow overriding the OpenAI endpoint, but many do
> (however these settings can sometimes be deeply hidden).


### API base / address / URL / ...

Typically, you should specify the API base in this format:  
`https://copilot.YOURSUBDOMAIN.workers.dev/v1`

However, some software may expect it without the `/v1` ending:  
`https://copilot.YOURSUBDOMAIN.workers.dev`

The relevant field may be labeled as "_OpenAI proxy_".
You might need to look under "_Advanced settings_" or similar sections.
Alternatively, it could be in some config file (check the relevant documentation for details).

For some command-line tools, you may need to set an environment variable, _e.g._:
```sh
set OPENAI_BASE_URL=https://copilot.YOURSUBDOMAIN.workers.dev/v1
```
_..or_:
```sh
set OPENAI_API_BASE=https://copilot.YOURSUBDOMAIN.workers.dev/v1
```

### API key

First you need to obtain a Copilot-enabled Github _access token_ 
(which should start either with `ghu_` or `gho_`).
Once deployed, this project provides a "/token" web page that assists in generating such a token.  
For alternative methods, please refer to the [`scripts`](scripts).

Input this token into the "_OpenAI API key_" field in your software settings.
Alternatively, it might be located in some config file
(check the relevant documentation for details).

For some command-line tools, you may need to set an environment variable, _e.g._:
```sh
set OPENAI_API_KEY=ghu_...
```


## Tech notes

Copilot provides following endpoints:
- `/v1/chat/completions`
  - `tools` and other related properties are ignored.
  - Output is post-processed in order to remove differences from OpenAI's.
    The `model` property always appears as "gpt-4".
- `/v1/embeddings`
  - [`input`](https://platform.openai.com/docs/api-reference/embeddings/create#embeddings-create-input):
    supports array of strings only
  - `encoding_format`, `dimensions`: ignored.
  - Output is not altered in any way, and has some minor differences from OpenAI's:
    - no `object` property (`"object": "embedding"`)
    - no `model` property (`"model": "text-embedding-3-small"`)
- `/v1/models`: Copilot does not provide models list, so it was determined empirically,
  and this API endpoint is served with `openai-github-copilot` itself.
- `/token`: serves a Web page that allows you to obtain a Github Copilot access token.
