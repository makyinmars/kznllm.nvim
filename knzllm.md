Directory Structure:
└── CONTRIBUTING.md
└── README.md
├── lua/
  ├── kznllm/
    └── buffer.lua
    ├── presets/
      └── basic.lua
    ├── specs/
      └── anthropic.lua
      └── init.lua
      └── openai.lua
    └── utils.lua
└── stylua.toml
├── templates/
  ├── anthropic/
    └── debug.xml.jinja
    └── fill_mode_system_prompt.xml.jinja
    └── fill_mode_user_prompt.xml.jinja
    └── long_context_documents.xml.jinja
  ├── deepseek/
    └── debug.xml.jinja
    └── fill_mode_system_prompt.xml.jinja
    └── fill_mode_user_prompt.xml.jinja
  ├── grok/
    └── fill_mode_system_prompt.xml.jinja
    └── fill_mode_user_prompt.xml.jinja
  ├── openai/
    └── debug.xml.jinja
    └── fill_mode_system_prompt.xml.jinja
    └── fill_mode_user_prompt.xml.jinja
  ├── qwen/
    └── fill_mode_instruct_completion_prompt.xml.jinja
    └── fill_mode_system_prompt.xml.jinja
    └── fill_mode_user_prompt.xml.jinja
  ├── vllm/
    └── debug.xml.jinja
    └── fill_mode_instruct_completion_prompt.xml.jinja
    └── fill_mode_system_prompt.xml.jinja
    └── fill_mode_user_prompt.xml.jinja

File Contents:

File: lua/kznllm/specs/init.lua
================================================
-- Preset configuration builder
---@class BasePresetConfig
---@field id string
---@field description string
---@field curl_options BaseProviderCurlOptions

---@class BaseProviderHeaderOptions
---@field endpoint string
---@field auth_format? string
---@field extra_headers? string[]

---@class BaseProviderCurlOptions: BaseProviderHeaderOptions
---@field data table

---@class BaseParameters
---@field model string

---@class BaseHeaders
---@field endpoint string
---@field auth_format? string
---@field extra_headers? string[]

---@class BasePresetBuilder
---@field debug_template_path? string
---@field provider BaseProvider
---@field config BasePresetConfig
---@field system_templates table[]
---@field message_templates table[]
---@field headers BaseHeaders
---@field params BaseParameters
---@field add_system_templates fun(self: BasePresetBuilder, templates: table)
---@field add_message_templates fun(self: BasePresetBuilder, templates: table)
---@field with_opts fun(self: BasePresetBuilder, params: table)
---@field build fun(self: BasePresetBuilder, args: table)

---@class BaseProvider
---@field api_key_name string
---@field base_url string
local BaseProvider = {}

---@class BaseProviderOptions
---@field api_key_name string
---@field base_url string

---@param opts BaseProviderOptions
---@return BaseProvider
function BaseProvider:new(opts)
  local instance = {
    api_key_name = opts.api_key_name,
    base_url = opts.base_url,
  }

  setmetatable(instance, { __index = self })
  return instance
end

---@param opts BaseProviderCurlOptions
---@return string[]
function BaseProvider:make_curl_args(opts)
  local url = self.base_url .. opts.endpoint

  local api_key = os.getenv(self.api_key_name)
  if not api_key then
    error(('ERROR: %s is missing from environment variables'):format(self.api_key_name))
  end

  -- stylua: ignore
  local args = {
    '-s', '--fail-with-body', '-N', --silent, with errors, unbuffered output
    '-X', 'POST',
    '-H', 'Content-Type: application/json',
    '-H', (opts.auth_format and opts.auth_format or 'Authorization: Bearer %s'):format(api_key),
  }

  if opts.extra_headers ~= nil then
    for _, header in ipairs(opts.extra_headers) do
      vim.list_extend(args, { '-H', header })
    end
  end

  vim.list_extend(args, { '-d', vim.json.encode(opts.data), url })
  return args
end

---@param line string
---@return string?
function BaseProvider.handle_sse_stream(line)
  error('handle_sse_stream NOT IMPLEMENTED', 1)
end

return BaseProvider


File: templates/qwen/fill_mode_user_prompt.xml.jinja
================================================
<|im_start|>user
{% if current_buffer_context or context_files %}
<|repo_name|>{{ current_buffer_context.path|default("current_file") if current_buffer_context else "workspace" }}

{% if current_buffer_context %}
<|file_sep|>{{ current_buffer_context.path }}
{{ current_buffer_context.text }}
{% endif %}

{% if context_files %}
{% for file in context_files %}
<|file_sep|>{{ file.path }}
{{ file.content }}
{% endfor %}
{% endif %}
{% endif %}

{% if visual_selection %}
{% if prefill %}
<|fim_prefix|>{{ visual_selection }}<|fim_suffix|>
RESPOND WITH RAW CODE ONLY - NO BACKTICKS - NO MARKDOWN - NO CODE BLOCK FORMATTING
<|fim_middle|>
{% else %}
Selected Code:
{{ visual_selection }}

RESPOND WITH RAW CODE ONLY - NO BACKTICKS - NO MARKDOWN - NO CODE BLOCK FORMATTING
{% endif %}
{% endif %}

Query: {{ user_query }}
RESPOND WITH RAW CODE ONLY - NO BACKTICKS - NO MARKDOWN - NO CODE BLOCK FORMATTING
<|im_end|>


File: templates/vllm/fill_mode_instruct_completion_prompt.xml.jinja
================================================
{{- bos_token }}
{%- if custom_tools is defined %}
 {%- set tools = custom_tools %}
{%- endif %}
{%- if not tools_in_user_message is defined %}
 {%- set tools_in_user_message = true %}
{%- endif %}
{%- if not date_string is defined %}
 {%- set date_string = \"26 Jul 2024\" %}
{%- endif %}
{%- if not tools is defined %}
 {%- set tools = none %}
{%- endif %}

{#- This block extracts the system message, so we can slot it into the right place. #}
{%- if messages[0]['role'] == 'system' %}
 {%- set system_message = messages[0]['content']|trim %}
 {%- set messages = messages[1:] %}
{%- else %}
 {%- set system_message = \"\" %}
{%- endif %}

{#- System message + builtin tools #}
{{- \"<|start_header_id|>system<|end_header_id|>\n\n\" }}
{%- if builtin_tools is defined or tools is not none %}
 {{- \"Environment: ipython\n\" }}
{%- endif %}
{%- if builtin_tools is defined %}
 {{- \"Tools: \" + builtin_tools | reject('equalto', 'code_interpreter') | join(\", \") + \"\n\n\"}}
{%- endif %}
{{- \"Cutting Knowledge Date: December 2023\n\" }}
{{- \"Today Date: \" + date_string + \"\n\n\" }}
{%- if tools is not none and not tools_in_user_message %}
 {{- \"You have access to the following functions. To call a function, please respond with JSON for a function call.\" }}
 {{- 'Respond in the format {\"name\": function name, \"parameters\": dictionary of argument name and its value}.' }}
 {{- \"Do not use variables.\n\n\" }}
 {%- for t in tools %}
 {{- t | tojson(indent=4) }}
 {{- \"\n\n\" }}
 {%- endfor %}
{%- endif %}
{{- system_message }}
{{- \"<|eot_id|>\" }}

{#- Custom tools are passed in a user message with some extra guidance #}
{%- if tools_in_user_message and not tools is none %}
 {#- Extract the first user message so we can plug it in here #}
 {%- if messages | length != 0 %}
 {%- set first_user_message = messages[0]['content']|trim %}
 {%- set messages = messages[1:] %}
 {%- else %}
 {{- raise_exception(\"Cannot put tools in the first user message when there's no first user message!\") }}
{%- endif %}
 {{- '<|start_header_id|>user<|end_header_id|>\n\n' -}}
 {{- \"Given the following functions, please respond with a JSON for a function call \" }}
 {{- \"with its proper arguments that best answers the given prompt.\n\n\" }}
 {{- 'Respond in the format {\"name\": function name, \"parameters\": dictionary of argument name and its value}.' }}
 {{- \"Do not use variables.\n\n\" }}
 {%- for t in tools %}
 {{- t | tojson(indent=4) }}
 {{- \"\n\n\" }}
 {%- endfor %}
 {{- first_user_message + \"<|eot_id|>\"}}
{%- endif %}

{%- for message in messages %}
 {%- if not (message.role == 'ipython' or message.role == 'tool' or 'tool_calls' in message) %}
 {{- '<|start_header_id|>' + message['role'] + '<|end_header_id|>\n\n'+ message['content'] | trim + '<|eot_id|>' }}
 {%- elif 'tool_calls' in message %}
 {%- if not message.tool_calls|length == 1 %}
 {{- raise_exception(\"This model only supports single tool-calls at once!\") }}
 {%- endif %}
 {%- set tool_call = message.tool_calls[0].function %}
 {%- if builtin_tools is defined and tool_call.name in builtin_tools %}
 {{- '<|start_header_id|>assistant<|end_header_id|>\n\n' -}}
 {{- \"<|python_tag|>\" + tool_call.name + \".call(\" }}
 {%- for arg_name, arg_val in tool_call.arguments | items %}
 {{- arg_name + '=\"' + arg_val + '\"' }}
 {%- if not loop.last %}
 {{- \", \" }}
 {%- endif %}
 {%- endfor %}
 {{- \")\" }}
 {%- else %}
 {{- '<|start_header_id|>assistant<|end_header_id|>\n\n' -}}
 {{- '{\"name\": \"' + tool_call.name + '\", ' }}
 {{- '\"parameters\": ' }}
 {{- tool_call.arguments | tojson }}
 {{- \"}\" }}
 {%- endif %}
 {%- if builtin_tools is defined %}
 {#- This means we're in ipython mode #}
 {{- \"<|eom_id|>\" }}
 {%- else %}
 {{- \"<|eot_id|>\" }}
 {%- endif %}
 {%- elif message.role == \"tool\" or message.role == \"ipython\" %}
 {{- \"<|start_header_id|>ipython<|end_header_id|>\n\n\" }}
 {%- if message.content is mapping or message.content is iterable %}
 {{- message.content | tojson }}
 {%- else %}
 {{- message.content }}
 {%- endif %}
 {{- \"<|eot_id|>\" }}
 {%- endif %}
{%- endfor %}
{%- if add_generation_prompt %}
 {{- '<|start_header_id|>assistant<|end_header_id|>\n\n' }}
{%- endif %}


File: templates/anthropic/debug.xml.jinja
================================================
model: {{data.model}}

system:
{% for context in data.system %}
{{ context.text | trim }}

cache_control: {{context.cache_control}}

---

{% endfor %}
{% for message in data.messages %}
{{message.role}}:

{% if message.content is iterable and (message.content is not string and message.content is not mapping) %}
{% for content in message.content %}
{{ content.text | trim }}
{% endfor %}
{% else %}
{{ message.content | trim }}
{% endif %}

{% endfor %}
---




File: templates/grok/fill_mode_system_prompt.xml.jinja
================================================
<|im_start|>system
You are a highly skilled code expert who provides raw code without any formatting or markup.

CRITICAL OUTPUT RULES:
1. NEVER use backticks (`) in any form
2. NEVER use code block delimiters (```)
3. NEVER use markdown formatting
4. Output ONLY raw code, exactly as it would appear in a code editor
5. No explanatory text unless specifically requested
6. No additional formatting or markup of any kind
7. When showing code examples, give them exactly as they would be written in the file
8. End response immediately after the code
9. No introduction or conclusion text

Example correct output:
def example():
    return "This is raw code"

Example incorrect output:
```python
def example():
    return "This is formatted"
```

Remember: The code should appear exactly as it would in a code editor - no additional formatting, no backticks, no markdown.
<|im_end|>


File: lua/kznllm/specs/anthropic.lua
================================================
local BaseProvider = require('kznllm.specs')
local utils = require('kznllm.utils')

local M = {}

---@class AnthropicProvider : BaseProvider
---@field make_curl_args fun(self, opts: AnthropicCurlOptions)
M.AnthropicProvider = {}

---@param opts? BaseProviderOptions
---@return AnthropicProvider
function M.AnthropicProvider:new(opts)
  -- Call parent constructor with base options
  local o = opts or {}
  local instance = BaseProvider:new({
    api_key_name = o.api_key_name or 'ANTHROPIC_API_KEY',
    base_url = o.base_url or 'https://api.anthropic.com',
  })

  -- Set proper metatable for inheritance
  setmetatable(instance, { __index = self })
  setmetatable(self, { __index = BaseProvider })

  ---silence lsp warning
  ---@type AnthropicProvider
  return instance
end

---
--- TYPE ANNOTATIONS
---

---@class AnthropicCurlOptions : AnthropicAPIHeaders
---@field data AnthropicAPIBody

---@class AnthropicAPIHeaders
---@field endpoint string
---@field auth_format? string
---@field extra_headers? string[]

---@class AnthropicAPIBody : AnthropicParameters, AnthropicPromptContext

---@class AnthropicPromptContext
---@field system AnthropicSystemContext[]
---@field messages AnthropicMessage[]

---@class AnthropicParameters
---@field model string | 'claude-3-5-sonnet-latest' | 'claude-3-5-haiku-latest'
---@field stop_sequences? string[]
---@field max_tokens? integer
---@field temperature? number
---@field top_k? integer
---@field top_p? number
---@field stream? boolean
---@field tool_choice? table
---@field tools? table[]

---@class AnthropicSystemContext
---@field type AnthropicSystemContentType
---@field text string
---@field cache_control? AnthropicCacheControl

---@alias AnthropicSystemContentType "text"
---@alias AnthropicMessageContentType "text" | "image" | "tool_use" | "tool_result" | "document"

---@class AnthropicCacheControl
---@field type "ephemeral"

---@alias AnthropicMessageRole "user" | "assistant"

---@class AnthropicMessageTextContent
---@field type "text"
---@field text string

---@class AnthropicMessage
---@field role AnthropicMessageRole
---@field content string | AnthropicMessageTextContent[]

---
--- DATA HANDLERS
---

local current_event_state

--- Anthropic SSE Specification
--- [See Documentation](https://docs.anthropic.com/en/api/messages-streaming#event-types)
---
--- Each server-sent event includes a named event type and associated JSON
--- data. Each event will use an SSE event name (e.g. event: message_stop),
--- and include the matching event type in its data.
---
--- Each stream uses the following event flow:
---
--- 1. `message_start`: contains a Message object with empty content.
---
--- 2. A series of content blocks, each of which have a `content_block_start`,
---    one or more `content_block_delta` events, and a `content_block_stop`
---    event. Each content block will have an index that corresponds to its
---    index in the final Message content array.
---
--- 3. One or more `message_delta` events, indicating top-level changes to the
---    final Message object.
--- 4. `message_stop` event
---
--- event types: `[message_start, content_block_start, content_block_delta, content_block_stop, message_delta, message_stop, error]`
---@param line string
---@return string?
function M.AnthropicProvider.handle_sse_stream(line)
  local content = ''
  for event, data in line:gmatch('event: ([%w_]+)\ndata: ({.-})\n') do
    if event == 'content_block_delta' then
      local json = vim.json.decode(data)
      if json.delta and json.delta.text then
        content = content .. json.delta.text
      end
    elseif event == 'content_block_start' then
    elseif event == 'content_block_stop' then
    elseif event == 'message_start' then
      vim.print(data)
    elseif event == 'message_stop' then
    elseif event == 'message_delta' then
    elseif event == 'ping' then
    elseif event == 'error' then
      vim.print(data)
    else
      vim.print(data)
    end
  end

  return content
end

---@class AnthropicPresetSystemTemplate
---@field type AnthropicSystemContentType
---@field path string
---@field cache_control? AnthropicCacheControl

---@class AnthropicPresetMessageTemplate
---@field role AnthropicMessageRole
---@field type AnthropicMessageContentType
---@field path string
---@field cache_control? AnthropicCacheControl

---@class AnthropicPresetBuilder : BasePresetBuilder
---@field provider AnthropicProvider
---@field debug_template? string
---@field system_templates AnthropicPresetSystemTemplate[]
---@field message_templates AnthropicPresetMessageTemplate[]
---@field headers AnthropicAPIHeaders
---@field params AnthropicParameters
M.AnthropicPresetBuilder = {}

local anthropic_template_path = utils.join_path({ utils.TEMPLATE_PATH, 'anthropic' })

---@param opts? { provider?: AnthropicProvider, debug_template_path?: string, headers?: AnthropicAPIHeaders, params?: AnthropicParameters }
---@return AnthropicPresetBuilder
function M.AnthropicPresetBuilder:new(opts)
  local o = opts or {}
  local instance = {
    debug_template_path = o.debug_template_path or utils.join_path({ anthropic_template_path, 'debug.xml.jinja' }),
    provider = o.provider or M.AnthropicProvider:new(),
    headers = o.headers or {
      endpoint = '/v1/messages',
      auth_format = 'x-api-key: %s',
      extra_headers = {
        'anthropic-version: 2023-06-01',
        'anthropic-beta: prompt-caching-2024-07-31',
      },
    },
    params = o.params or {
      ['model'] = 'claude-3-5-sonnet-20241022',
      ['stream'] = true,
      ['max_tokens'] = 8192,
      ['temperature'] = 0.7,
    },
    system_templates = {},
    message_templates = {},
  }
  setmetatable(instance, { __index = self })
  return instance
end

---@param opts { params: AnthropicParameters, headers: AnthropicAPIHeaders, provider: AnthropicProvider }
function M.AnthropicPresetBuilder:with_opts(opts)
  local cpy = vim.deepcopy(self)
  for k, v in pairs(opts) do
    cpy[k] = v
  end
  return cpy
end

--- Mutates the builder's system templates
---@param system_templates AnthropicPresetSystemTemplate[]
function M.AnthropicPresetBuilder:add_system_prompts(system_templates)
  for _, template in ipairs(system_templates) do
    table.insert(self.system_templates, template)
  end
  return self
end

--- Mutates the builder's message templates
---@param message_templates AnthropicPresetMessageTemplate[]
function M.AnthropicPresetBuilder:add_message_prompts(message_templates)
  for _, template in ipairs(message_templates) do
    table.insert(self.message_templates, template)
  end
  return self
end

---Renders all templates and builds curl args in the correct format according to Anthropic API spec
---@return AnthropicCurlOptions
function M.AnthropicPresetBuilder:build(args)
  ---@type AnthropicSystemContext[]
  local system = {}

  for _, template in ipairs(self.system_templates) do
    if template.type == 'text' then
      table.insert(system, {
        type = template.type,
        text = utils.make_prompt_from_template({ template_path = template.path, prompt_args = args }),
        cache_control = template.cache_control,
      })
    end
  end

  ---@type AnthropicMessage[]
  local messages = {}

  for _, template in ipairs(self.message_templates) do
    if template.type == 'text' then
      local message_content = {
        type = 'text',
        text = utils.make_prompt_from_template({ template_path = template.path, prompt_args = args }),
        cache_control = template.cache_control,
      }
      table.insert(messages, {
        role = template.role,
        content = { message_content },
      })
    end
  end

  return vim.tbl_extend('keep', self.headers, {
    data = vim.tbl_extend('keep', self.params, { system = system, messages = messages }),
  })
end

return M


File: templates/qwen/fill_mode_system_prompt.xml.jinja
================================================
<|im_start|>system
{% if replace %}
You are in CODE ONLY MODE. Critical rules:
- NEVER USE MARKDOWN CODE BLOCKS
- NEVER USE BACKTICKS IN ANY FORM
- NEVER ADD CODE FORMATTING SYMBOLS
- OUTPUT RAW CODE ONLY
- NO TEXT BEFORE OR AFTER CODE
- NO EXPLANATIONS
- REMOVE INSTRUCTION COMMENTS
- PRESERVE NON-INSTRUCTION COMMENTS
{% else %}
You are in STRICT CODE MODE:
- NEVER USE MARKDOWN CODE BLOCKS
- NEVER USE BACKTICKS IN ANY FORM
- NEVER ADD CODE FORMATTING SYMBOLS
- OUTPUT RAW CODE ONLY
- NO EXPLANATIONS UNLESS EXPLICITLY REQUESTED
{% endif %}

{% if tools %}
Available Tools:
<tools>
{%- for tool in tools %}
{"type": "function", "function": {{ tool | tojson }}}
{%- endfor %}
</tools>
{% endif %}
<|im_end|>


File: templates/openai/debug.xml.jinja
================================================
model: {{data.model}}

{% for message in data.messages %}
{{message.role}}:

{% if message.content is iterable and (message.content is not string and message.content is not mapping) %}
{% for content in message.content %}
{{ content.text | trim }}
{% endfor %}
{% else %}
{{ message.content | trim }}
{% endif %}

{% endfor %}
---




File: templates/anthropic/long_context_documents.xml.jinja
================================================
First, let's establish the context of related documents:

<related_documents>
{% for file in context_files %}
<document index="{{loop.index}}">
<path>{{file.path}}</path>
<content>
{{file.content | trim}}
</content>
</document>
{% endfor %}
</related_documents>



File: lua/kznllm/utils.lua
================================================
local api = vim.api

local M = {}

function M.join_path(paths, path_sep)
  path_sep = path_sep or '/'
  return table.concat(paths, path_sep)
end

-- NOTE: this is a relative path meant to point at the template directory
M.PLUGIN_PATH = vim.fn.fnamemodify(debug.getinfo(1, 'S').source:sub(2), ':h:h:h')
M.TEMPLATE_PATH = M.join_path({ M.PLUGIN_PATH, 'templates' })

--
-- [ CONTEXT BUILDING UTILITY FUNCTIONS ]
--

---Handles visual selection depending on the specified mode and some expected states of the user's current buffer.
--- Returns the selection and whether or not text was replaced
---
---@param opts { debug: boolean? } optional values including debug mode
---@return string selection
---@return boolean replace
function M.get_visual_selection(opts)
  local mode = api.nvim_get_mode().mode

  -- get visual selection and current cursor position (1-indexed)
  local _, srow, scol = unpack(vim.fn.getpos('v'))
  local _, erow, ecol = unpack(vim.fn.getpos('.'))

  -- normalize start + end such that start_pos < end_pos and converts to 0-index
  srow, scol, erow, ecol = srow - 1, scol - 1, erow - 1, ecol - 1
  if srow > erow or (srow == erow and scol > ecol) then
    srow, erow, scol, ecol = erow, srow, ecol, scol
  end

  -- in visual block and visual line mode, we expect first column of srow and last column of erow
  if mode == 'V' or mode == '\22' or mode == 'n' then
    scol, ecol = 0, -1
  else
    local erow_content = vim.api.nvim_buf_get_lines(0, erow, erow + 1, false)[1]
    if ecol < #erow_content then
      ecol = ecol + 1
    end
  end

  -- handling + cleanup for visual selection
  local visual_selection
  local replace_mode = not (mode == 'n')

  if replace_mode then
    api.nvim_feedkeys(vim.api.nvim_replace_termcodes('<Esc>', false, true, true), 'nx', false)
    visual_selection = table.concat(api.nvim_buf_get_text(0, srow, scol, erow, ecol, {}), '\n')
  end

  -- clear the visual selection depending on condition
  local debug = opts and opts.debug
  if not debug and replace_mode then
    api.nvim_buf_set_text(0, srow, scol, erow, ecol, {})
  end

  return visual_selection, replace_mode
end

---project scoped context
---
---Retrieves project files based on the context directory identifier in the current working directory.
---
---@return { path: string, content: string }? context_files list of files in the context directory
function M.get_project_files()
  if vim.fn.executable('fd') ~= 1 then
    -- only use project mode if `fd` is available
    return
  end

  local fd_dir_result = vim.system({ 'fd', '-td', '-HI', '.kzn', '-1' }):wait()
  local context_dir = vim.trim(fd_dir_result.stdout)

  -- do not respect `.gitignore`, look for hidden
  local fd_files_result = vim.system({ 'fd', '-tf', '-L', '.', context_dir }):wait()
  local files = {}
  for file in vim.gsplit(fd_files_result.stdout, '\n', { plain = true, trimempty = true }) do
    local content = vim.fn.readfile(file)
    if #content > 0 then
      table.insert(files, { path = file, content = table.concat(content, '\n') })
    end
  end

  return files
end

---Creates a prompt from template
---@param opts { template_path: string, prompt_args: table }
---@return string
function M.make_prompt_from_template(opts)
  if vim.fn.executable('minijinja-cli') ~= 1 then
    error("Can't find minijinja-cli, download it from https://github.com/mitsuhiko/minijinja or add it to $PATH", 1)
  end

  local prompt_template_path = opts.template_path
  local json_data = vim.json.encode(opts.prompt_args)

  local active_job = vim
      .system(
        { 'minijinja-cli', '-f', 'json', '--lstrip-blocks', '--trim-blocks', prompt_template_path, '-' },
        { stdin = json_data }
      )
      :wait()

  if active_job.code ~= 0 then
    error('[minijinja-cli] (exit code: ' .. active_job.code .. ')\n' .. active_job.stderr, vim.log.levels.ERROR)
  end

  return active_job.stdout
end

return M


File: templates/deepseek/fill_mode_user_prompt.xml.jinja
================================================
{% if current_buffer_context or context_files %}
CONTEXT:
{% endif %}
{% if current_buffer_context.path %}
Document:0
Title: {{ current_buffer_context.path }}
{% endif %}
{% if current_buffer_context %}
Content:
```{{ current_buffer_context.filetype }}
{{ current_buffer_context.text }}
```
{% endif %}
{% if context_files %}
  {% for file in context_files %}
Document:{{ loop.index }}
Title: {{ file.path }}
Content:
```
{{ file.content }}
```
  {% endfor %}
{% endif %}
{% if visual_selection %}
Code:
```{{ current_buffer_filetype }}
{{ visual_selection }}
```

  {% if prefill %}
INSTRUCTION: Replace the code in the block given above. ONLY return the valid code fragment surrounded by a code fence with backticks.
  {% else %}
INSTRUCTION: Replace the code in the block given above. ONLY return the code fragment that is requested in the code snippet WITHOUT backticks. DO NOT surround the code fragment in backticks.
  {% endif %}
{% endif %}
QUERY: {{ user_query }}


File: templates/vllm/debug.xml.jinja
================================================
model: {{data.model}}

{% for message in data.messages %}
{{message.role}}:

{% if message.content is iterable and (message.content is not string and message.content is not mapping) %}
{% for content in message.content %}
{{ content.text | trim }}
{% endfor %}
{% else %}
{{ message.content | trim }}
{% endif %}

{% endfor %}
---




File: templates/anthropic/fill_mode_user_prompt.xml.jinja
================================================
{% if current_buffer_context.text %}
Let's establish the context of the current file:

<current_file_context>
<path>{{current_buffer_context.path}}</path>
<filetype>{{current_buffer_context.filetype}}</filetype>
<content>
{{current_buffer_context.text | trim}}
</content>
</current_file_context>

{% endif %}
{% if visual_selection %}
Now, here's the specific code snippet you need to work on:

<code_snippet>
<language>{{current_buffer_context.filetype}}</language>
<selected_code>
{{visual_selection | trim}}
</selected_code>
</code_snippet>

{% endif %}
{% if replace %}
Here's the user's specific query or instruction:
<user_query>
{{user_query | trim}}
</user_query>
{% else %}
Here's the user's specific query or instruction:
<user_query>
{{user_query | trim}}
</user_query>

<instructions>
Your task is to improve this code snippet by following these steps:

1. Analyze the code:
   a. Identify and list syntax errors
   b. Identify and list logical errors
   c. Identify and list unimplemented functionality
   d. Consider and note potential edge cases or unintended consequences
   e. Identify and list bad coding practices that need correction
   f. Focus on the selected code and its immediate context
   g. Consider how changes will affect the broader context of the file

2. Plan your changes:
   - Determine the most efficient approach to fix errors and implement missing features
   - Ensure your planned changes maintain the original intent and the user's unique coding style
   - Consider how your changes might affect performance and readability
   - Avoid introducing unnecessary dependencies or complexity
   - Explicitly consider the performance implications of planned changes

3. Implement the improvements:
   - Fix all identified errors
   - Implement any missing functionality described in the comments
   - Maintain the original code style and formatting unless it directly contributes to the errors
   - Keep relevant comments, including those used to guide your changes
   - If you make significant changes, add brief, clear comments explaining your modifications

4. Review and finalize:
   - Ensure your changes have addressed all issues and requirements
   - Verify that the final code is valid and ready to run in the specified language
   - Check that your changes don't introduce new problems or unintended side effects
   - Consider how your changes might affect the broader context of the file

Wrap your analysis inside <code_review> tags, showing your thought process before implementing the changes. Then, wrap the corrected and implemented code inside <corrected_code> tags.

Remember:
- Only output valid code without any backticks or surrounding markdown
- Do not add features or functionality beyond what is explicitly mentioned in the code or comments
- Focus on substituting the user's selection with a valid code fragment, considering the given context
- Provide thoughtful but brief and direct feedback
- Speak as an expert in low-level concepts, identifying and correcting bad practices while preserving the user's unique style
</instructions>

Now, please proceed with your analysis and code correction.
{% endif %}



File: templates/vllm/fill_mode_user_prompt.xml.jinja
================================================
{% if current_buffer_context or context_files %}
CONTEXT:
{% endif %}
{% if current_buffer_context.path %}
Document:0
Title: {{ current_buffer_context.path }}
{% endif %}
{% if current_buffer_context %}
Content:
```{{ current_buffer_context.filetype }}
{{ current_buffer_context.text }}
```
{% endif %}
{% if context_files %}
  {% for file in context_files %}
Document:{{ loop.index }}
Title: {{ file.path }}
Content:
```
{{ file.content }}
```
  {% endfor %}
{% endif %}
{% if visual_selection %}
Code:
```{{ current_buffer_filetype }}
{{ visual_selection }}
```

  {% if prefill %}
INSTRUCTION: Replace the code in the block given above. ONLY return the valid code fragment surrounded by a code fence with backticks.
  {% else %}
INSTRUCTION: Replace the code in the block given above. ONLY return the code fragment that is requested in the code snippet WITHOUT backticks. DO NOT surround the code fragment in backticks.
  {% endif %}
{% endif %}
QUERY: {{ user_query }}


File: CONTRIBUTING.md
================================================
For developing any kind of plugin using `lazy.nvim` you basically just need to do this:
- clone the repo into your workspace so that lua LSP can find dependencies (this should be your config directory if you want type hints to pass through properly)
- point the plugin to local directory
- update the plugin using `:Lazy` and confirm it's looking at the local directory

For development, you want to install the plugin locally and update your lazy config like this (same as the main project README with `dev = true` and `dir = path/to/plugin`):

```lua
{
  'chottolabs/kznllm.nvim',
  dev = true,
  dir = '$HOME/.config/nvim/plugins/kznllm.nvim',
  dependencies = {
    { 'nvim-lua/plenary.nvim' },
    -- { 'chottolabs/plenary.nvim' }, -- patched to resolve symlinked directories
  },
  config = function(self)
  ...
  end
},
```

# Overview

`kznllm` (since `v0.2`) at its core provides:

1. a set of utility functions for pulling context from files on disk and/or your active nvim instance.
2. a minimal implementation of specs from API providers that are relevant to helping you stream tokens into a nvim buffer

If you take a look at `lua/kznllm/presets.lua` you can learn how I built out the entire feature set of kznllm in `v0.1` by basically stitching together utility functions from the core library and telling neovim what you want to do with it (i.e. open up scratch buffers, specify extmark position, etc.)

You don't have to use anything from presets at all, it's just there to provide a starting point for making the plugin functional. It can be relatively simple to add new features/capabilities for example:

- a workflow for interactively cycling through a dataset and triggering an LLM evaluation
- add an intermediate API call to a cheap/fast model for a long-context "project mode"
- ask an slow/expensive model to freely output a response and pipe it into a cheap/fast model for structured output

## Prompt Templates

The "no visual selection mode" is really just a "no replace" mode controlled by `local replace_mode = not (mode == 'n')`.

If you look at the system prompts, it's just defining all the logic in the same template, you can add whatever arguments you want in this to suit your use case:

```j2
{%- if replace -%}
You should replace the code that you are sent, only following the comments. Do not talk at all. Only output valid code. Do not provide any backticks that surround the code. Never ever output backticks like this ```. Any comment that is asking you for something should be removed after you satisfy them. Other comments should left alone. Do not output backticks
{%- else -%}
You are a Senior Engineer at a Fortune 500 Company. You will be provided with code samples, academic papers, and documentation as supporting context to assist you in answering user queries about coding. Your task is to analyze this information and use it to provide accurate, helpful responses to the user's coding-related questions.
{%- endif -%}
```

An interesting thing you might consider is implementing a "project-scoped" template directory that can look for documentation files and pipe it into the args (you can do this with the actual text or the file path `{% include <absolute_file_path> %}`). `minijinja-cli` makes this kind of stuff super easy to do.

# How it works

The plugin tries to do as little *magic* as possible, so the abstraction boundaries are relatively transparent.

From looking at a few key chunks of code, you should get a solid understanding of how you might want to structure things (treat `invoke_llm` as a simple reference, you can come up with your own structure for a feature)

the call to `invoke_llm` in `init.lua`

```lua
...
presets.invoke_llm(
  SELECTED_PRESET.make_data_fn,
  spec.make_curl_args,
  spec.make_job,
  vim.tbl_extend('keep', SELECTED_PRESET.opts, {
    template_directory = TEMPLATE_DIRECTORY,
  })
)
...
```

Here you see that `make_curl_args` and `make_job` was taken straight from the spec definitions - you can expect that these don't change very often and you likely won't need to touch it... however you can just as easily pass your own implementation into it.

Notice that `make_data_fn` comes from a preset - this is where you construct the "data" portion of your API call. The structure varies greatly between API providers and you will often find that you can't just plug in the same prompt across multiple providers and expect the same performance (e.g. take advantage of prefilling in Anthropic or chat vs. completion mode)

Now take a look at `lua/kznllm/presets.lua`

```lua
{
  id = 'chat-model',
  provider = 'groq',
  make_data_fn = make_data_for_openai_chat,
  opts = {
    model = 'llama-3.1-70b-versatile',
    data_params = {
      max_tokens = 8192,
      temperature = 0.7,
    },
    debug_fn = openai_debug_fn,
    base_url = 'https://api.groq.com',
    endpoint = '/openai/v1/chat/completions',
  },
},
```

Here you'll see how we've bundled together all of the things we need - you'll also notice some of the weirder things I've done which is support a custom `debug_fn` which specifies how to parse out the generated data and write it out to a scratch buffer.

In opts, we can pass anything into it that's supported by the model and it will just get passed into `vim.json.encode` at the end


File: README.md
================================================
# kznllm.nvim

Built for a single keybind `leader + k`, it does nothing more than fill in some LLM completion into the text buffer. It has two main behaviors:
1. If you made a visual selection, it will attempt to replace your selection with a valid code fragment. 
2. If you make no visual selection, it can yap freely (or do something else specified by a good template).

> [!NOTE]
> `kznllm.nvim` is compatible with Neovim 0.10.1 or later (uses `vim.system`)
> project-mode is also available when you have `sharkdp/fd` installed and a directory named `.kzn`. It will check your current working directory for a `.kzn`.

It's easy to hack on and implement customize behaviors without understanding much about nvim plugins. Try the default preset configuration provided below, but I recommend you fork the repo and using the preset as a reference for implementing your own features.

- **close-to-natty** coding experience
- add custom prompt templates
- pipe any context into template engine
- extend with custom features/modes

https://github.com/user-attachments/assets/406fc75f-c204-42ec-80a0-0f9e186c34c7

## Configuration

Make your API keys available via environment variables
```
export LAMBDA_API_KEY=secret_...
export ANTHROPIC_API_KEY=sk-...
export OPENAI_API_KEY=sk-proj-...
export GROQ_API_KEY=gsk_...
export DEEPSEEK_API_KEY=vllm_...
export VLLM_API_KEY=vllm_...
```

> [!NOTE]
> This plugin depends on [minijinja-cli](https://github.com/mitsuhiko/minijinja) (`cargo install minijinja-cli`, but double-check) - way easier to compose prompts.

full config w/ supported presets and a switch mechanism and provider-specific debug functions

```lua
{
  'chottolabs/kznllm.nvim',
  dependencies = {
    { 'j-hui/fidget.nvim' },
  },
  config = function(self)
    local presets = require 'kznllm.presets.basic'

    vim.keymap.set({ 'n', 'v' }, '<leader>m', function()
      presets.switch_presets(presets.options)
    end, { desc = 'switch between presets' })

    local function invoke_with_opts(opts)
      return function()
        local preset = presets.load_selected_preset(presets.options)
        preset.invoke(opts)
      end
    end

    vim.keymap.set({ 'n', 'v' }, '<leader>K', invoke_with_opts { debug = true },
      { desc = 'Send current selection to LLM debug' })
    vim.keymap.set({ 'n', 'v' }, '<leader>k', invoke_with_opts { debug = false },
      { desc = 'Send current selection to LLM llm_fill' })

    vim.api.nvim_set_keymap('n', '<Esc>', '', {
      noremap = true,
      silent = true,
      callback = function()
        vim.api.nvim_exec_autocmds('User', { pattern = 'LLM_Escape' })
      end,
    })
  end,
},
```

for local openai server (e.g. `vllm serve` w/ `--api-key <token>` and `--served-model-name meta-llama/Meta-Llama-3.1-8B-Instruct`) set `VLLM_API_KEY=<token>`

---

## Contributing

See [CONTRIBUTING](CONTRIBUTING.md) to understand the typical development workflow for Neovim plugins using `Lazy` and some straightforward ways you can modify the plugin to suit your needs

---

## Custom Progress Updates for your Amusement

`init.lua`
```lua
local function yap_generator()
  math.randomseed(os.time())
  local yap_cycle = {
    "yapped for %ds",
    "...",
  }

  local idx = math.random(1, #yap_cycle)
  return function()
    idx = idx + 1
    if idx > #yap_cycle then
      idx = 1
    end
    return yap_cycle[idx]
  end
end

local yap = yap_generator()

local function progress_fn(state)
  local now = os.time()
  if (now ~= state.last_updated) and ((now - state.start) % 3) == 0 then
    state.last_updated = now
    return yap()
  end
end

vim.keymap.set( { "n", "v" }, "<leader>K", invoke_with_opts({ debug = true, progress_message_fn = progress_fn }), { desc = "..." })
vim.keymap.set( { "n", "v" }, "<leader>k", invoke_with_opts({ debug = false, progress_message_fn = progress_fn }), { desc = "..." })
```

---

## Additional Notes

Originally based on [dingllm.nvim](https://github.com/yacineMTB/dingllm.nvim) - but diverged quite a bit

- prompts user for additional context before filling
- structured to make the inherent coupling between neovim logic, LLM streaming spec, and model-specific templates more explicit
- uses jinja as templating engine for ensuring correctness in more complex prompts
- preset defaults + simple approach for overriding them
- free cursor movement during generation
- avoids "undojoin after undo" error


File: templates/grok/fill_mode_user_prompt.xml.jinja
================================================
<|im_start|>user
{% if current_buffer_context or context_files %}
<|repo_name|>{{ current_buffer_context.path|default("current_file") if current_buffer_context else "workspace" }}

{% if current_buffer_context %}
<|file_sep|>{{ current_buffer_context.path }}
{{ current_buffer_context.text }}
{% endif %}

{% if context_files %}
{% for file in context_files %}
<|file_sep|>{{ file.path }}
{{ file.content }}
{% endfor %}
{% endif %}
{% endif %}

{% if visual_selection %}
{% if prefill %}
<|fim_prefix|>{{ visual_selection }}<|fim_suffix|>
PROVIDE CODE EXACTLY AS IT WOULD APPEAR IN A CODE EDITOR:
- NO backticks (`)
- NO code blocks (```)
- NO markdown
- NO formatting
- NO explanations
<|fim_middle|>
{% else %}
Selected Code:
{{ visual_selection }}

PROVIDE CODE EXACTLY AS IT WOULD APPEAR IN A CODE EDITOR:
- NO backticks (`)
- NO code blocks (```)
- NO markdown
- NO formatting
- NO explanations
{% endif %}
{% endif %}

Query: {{ user_query }}

IMPORTANT: Output the code exactly as it would appear in a code editor - no backticks, no code blocks, no markdown, no explanations.
<|im_end|>
<|im_start|>assistant
I understand. I will provide raw code only, exactly as it would appear in a code editor, without any backticks, code blocks, markdown formatting, or explanations.
<|im_end|>


File: lua/kznllm/specs/openai.lua
================================================
local BaseProvider = require('kznllm.specs')
local utils = require('kznllm.utils')

local M = {}

---@class OpenAIProvider : BaseProvider
---@field make_curl_args fun(self, opts: OpenAICurlOptions)
M.OpenAIProvider = {}

---@param opts? BaseProviderOptions
---@return OpenAIProvider
function M.OpenAIProvider:new(opts)
  -- Call parent constructor with base options

  local o = opts or {}
  local instance = BaseProvider:new({
    api_key_name = o.api_key_name or 'OPENAI_API_KEY',
    base_url = o.base_url or 'https://api.openai.com',
  })

  -- Set proper metatable for inheritance
  setmetatable(instance, { __index = self })
  setmetatable(self, { __index = BaseProvider })

  ---silence lsp warning
  ---@type OpenAIProvider
  return instance
end

---
--- TYPE ANNOTATIONS
---

---@class OpenAICurlOptions : OpenAIHeaders
---@field data OpenAIBody

---@class OpenAIHeaders
---@field endpoint string
---@field auth_format? string
---@field extra_headers? string[]

---@class OpenAIBody : OpenAIParameters, OpenAIPromptContext

---@class OpenAIPromptContext
---@field messages OpenAIMessage[]

---@class OpenAIParameters
---@field model string
---@field max_tokens? integer
---@field max_completion_tokens? integer
---@field temperature? number
---@field top_p? number
---@field frequency_penalty? number
---@field presence_penalty? number

---@alias OpenAIMessageRole "system" | "user" | "assistant"
---@class OpenAIMessage
---@field role OpenAIMessageRole
---@field content string | OpenAIMessageContent[]

---@alias OpenAIMessageContentType "text" | "image"
---@class OpenAIMessageContent
---@field type OpenAIMessageContentType
---@field text string

--- Process server-sent events based on OpenAI spec
--- [See Documentation](https://platform.openai.com/docs/api-reference/chat/create#chat-create-stream)
---
---@param buf string
---@return string
function M.OpenAIProvider.handle_sse_stream(buf)
  -- based on sse spec (OpenAI spec uses data-only server-sent events)
  local content = ''

  for data in buf:gmatch('data: ({.-})\n') do
    -- if data and data:match '"delta":' then
    local json = vim.json.decode(data)
    -- sglang server returns the role as one of the events and it becomes `vim.NIL`, so we have to handle it here
    if json.choices and json.choices[1] and json.choices[1].delta and json.choices[1].delta.content then
      content = content .. json.choices[1].delta.content
    else
      vim.print(data)
    end
    -- end
  end

  return content
end

---@class OpenAIPresetConfig
---@field id string
---@field description string
---@field curl_options OpenAICurlOptions

---@class OpenAIPresetSystemTemplate
---@field path string

---@class OpenAIPresetMessageTemplate
---@field type OpenAIMessageContentType
---@field role OpenAIMessageRole
---@field path string

---@class OpenAIPresetBuilder : BasePresetBuilder
---@field provider OpenAIProvider
---@field system_templates OpenAIPresetSystemTemplate[]
---@field message_templates OpenAIPresetMessageTemplate[]
---@field debug_template? string
---@field headers OpenAIHeaders
---@field params OpenAIParameters
M.OpenAIPresetBuilder = {}

local openai_template_path = utils.join_path({ utils.TEMPLATE_PATH, 'openai' })

---@param opts? { provider: OpenAIProvider, headers: OpenAIHeaders, params: OpenAIParameters, debug_template_path: string }
---@return OpenAIPresetBuilder
function M.OpenAIPresetBuilder:new(opts)
  local o = opts or {}
  local instance = {
    provider = o.provider or M.OpenAIProvider:new(),
    debug_template_path = o.debug_template_path or utils.join_path({ openai_template_path, 'debug.xml.jinja' }),
    headers = o.headers or { endpoint = '/v1/chat/completions' },
    params = (opts and opts.params) and opts.params or {
      ['model'] = 'o1-mini',
      ['stream'] = true,
    },
    system_templates = {},
    message_templates = {},
  }
  setmetatable(instance, { __index = self })
  return instance
end

---@param opts { params: OpenAIParameters, headers: OpenAIHeaders, provider: OpenAIProvider }
function M.OpenAIPresetBuilder:with_opts(opts)
  local cpy = vim.deepcopy(self)
  for k, v in pairs(opts) do
    cpy[k] = v
  end
  return cpy
end

---@param system_templates OpenAIPresetSystemTemplate[]
function M.OpenAIPresetBuilder:add_system_prompts(system_templates)
  for _, template in ipairs(system_templates) do
    table.insert(self.system_templates, 1, template)
  end
  return self
end

---@param message_templates OpenAIPresetMessageTemplate[]
function M.OpenAIPresetBuilder:add_message_prompts(message_templates)
  for _, template in ipairs(message_templates) do
    table.insert(self.message_templates, template)
  end
  return self
end

---@return OpenAICurlOptions
function M.OpenAIPresetBuilder:build(args)
  ---@type OpenAIMessage[]
  local messages = {}

  for _, template in ipairs(self.system_templates) do
    table.insert(messages, {
      role = 'system',
      content = utils.make_prompt_from_template({ template_path = template.path, prompt_args = args }),
    })
  end

  for _, template in ipairs(self.message_templates) do
    if template.type == 'text' then
      local message_content = {
        type = template.type,
        text = utils.make_prompt_from_template({ template_path = template.path, prompt_args = args }),
      }

      table.insert(messages, {
        role = template.role,
        content = { message_content },
      })
    end
  end

  return vim.tbl_extend('keep', self.headers, {
    data = vim.tbl_extend('keep', self.params, { messages = messages }),
  })
end

return M


File: templates/openai/fill_mode_user_prompt.xml.jinja
================================================
{% if current_buffer_context %}
{% if current_buffer_context.text or context_files %}
CONTEXT:
{% endif %}
{% if current_buffer_context.text %}
Document:0
Title: {{ current_buffer_context.path }}
{% endif %}
{% if current_buffer_context.text %}
Content:
{{ current_buffer_context.text }}
{% endif %}
{% endif %}
{% if context_files %}
<related_documents>
{% for file in context_files %}
<document index="{{loop.index}}">
<path>{{file.path}}</path>
<content>
{{file.content | trim}}
</content>
</document>
{% endfor %}
</related_documents>
{% endif %}
{% if visual_selection %}
Code:
{{ visual_selection }}

{% endif %}
{% if replace %}
INSTRUCTION: Replace the code in the block given above. ONLY return the code fragment that is requested in the code snippet WITHOUT backticks. DO NOT surround the code fragment in backticks.
{% endif %}
QUERY: {{ user_query }}



File: stylua.toml
================================================
column_width = 120
indent_type = "Spaces"
indent_width = 2
call_parentheses = "Input"

quote_style = "AutoPreferSingle"
collapse_simple_statement = "Never"


File: templates/deepseek/fill_mode_system_prompt.xml.jinja
================================================
{% if replace %}
You should replace the code that you are sent, only following the comments. Do not talk at all. Only output valid code. Do not provide any backticks that surround the code. Never ever output backticks like this ```. Any comment that is asking you for something should be removed after you satisfy them. Other comments should left alone. Do not output backticks
{% else %}
You are a Senior Engineer at a Fortune 500 Company. You will be provided with code samples, academic papers, and documentation as supporting context to assist you in answering user queries about coding. Your task is to analyze this information and use it to provide accurate, helpful responses to the user's coding-related questions.
{% endif %}


File: lua/kznllm/presets/basic.lua
================================================
local utils = require('kznllm.utils')
local buffer_manager = require('kznllm.buffer').buffer_manager
local api = vim.api

local anthropic = require('kznllm.specs.anthropic')
local openai = require('kznllm.specs.openai')

local progress = require('fidget.progress')

local M = {}

---@class BaseTask
---@field id string
---@field description string
---@field invoke fun(opts: { debug: boolean, progress_fn: fun(state) })

---@param config { id: string, description: string, preset_builder: OpenAIPresetBuilder | AnthropicPresetBuilder }
local function NewBaseTask(config)
  return {
    id = config.id,
    description = config.description,
    invoke = function(opts)
      vim.ui.input({ prompt = 'prompt: ' }, function(user_query)
        local selection, replace = utils.get_visual_selection(opts)

        local current_buf_id = api.nvim_get_current_buf()
        local current_buffer_context = buffer_manager:get_buffer_context(current_buf_id)

        local p = progress.handle.create({
          title = ('[%s]'):format(replace and 'replacing' or 'yapping'),
          lsp_client = { name = 'kznllm' },
        })

        local prompt_args = {
          user_query = user_query,
          visual_selection = selection,
          current_buffer_context = current_buffer_context,
          replace = replace,
          context_files = utils.get_project_files(),
        }

        local curl_options = config.preset_builder:build(prompt_args)

        if opts.debug then
          local scratch_buf_id = buffer_manager:create_scratch_buffer()
          local debug_data = utils.make_prompt_from_template({
            template_path = config.preset_builder.debug_template_path,
            prompt_args = curl_options,
          })

          buffer_manager:write_content(debug_data, scratch_buf_id)
          vim.cmd('normal! Gzz')
        end

        local provider = config.preset_builder.provider
        local args = provider:make_curl_args(curl_options)

        local state = { start = os.time(), last_updated = nil }
        p:report({ message = ('%s'):format(config.description) })
        local message_fn = opts.progress_message_fn and opts.progress_message_fn
            or function(s)
              return 'yapped'
            end
        local message = message_fn(state)
        local _ = buffer_manager:create_streaming_job(args, provider.handle_sse_stream, function()
          local progress_message = message_fn(state)
          if progress_message ~= nil then
            message = progress_message
          end

          local elapsed = os.time() - state.start
          if message:format(elapsed) ~= message then
            p:report({ message = message:format(os.time() - state.start) })
          end
        end, function()
          p:finish()
        end)
      end)
    end,
  }
end

---@param preset_list BaseTask[]
function M.switch_presets(preset_list)
  local preset_idx = vim.g.PRESET_IDX and math.min(vim.g.PRESET_IDX, #preset_list) or 1
  local selected_preset = preset_list[preset_idx]

  vim.ui.select(preset_list, {
    format_item = function(item)
      return ('%-30s %40s'):format(item.id .. (item == selected_preset and ' *' or '  '), item.description)
    end,
  }, function(choice, idx)
    if choice then
      vim.g.PRESET_IDX = idx
      selected_preset = preset_list[idx]
    end
    print(('%-30s %40s'):format(selected_preset.id, selected_preset.description))
  end)
end

function M.load_selected_preset(preset_list)
  local idx = vim.g.PRESET_IDX or 1
  local preset = preset_list[idx]

  return preset
end

local anthropic_template_path = utils.join_path({ utils.TEMPLATE_PATH, 'anthropic' })
local anthropic_system_template = utils.join_path({ anthropic_template_path, 'fill_mode_system_prompt.xml.jinja' })
local anthropic_user_template = utils.join_path({ anthropic_template_path, 'fill_mode_user_prompt.xml.jinja' })

local BasicAnthropicPreset = anthropic.AnthropicPresetBuilder
    :new()
    :add_system_prompts({
      {
        type = 'text',
        path = anthropic_system_template,
        cache_control = { type = 'ephemeral' },
      },
    })
    :add_message_prompts({
      { type = 'text', role = 'user', path = anthropic_user_template },
    })

local openai_template_path = utils.join_path({ utils.TEMPLATE_PATH, 'openai' })
local openai_system_template = utils.join_path({ openai_template_path, 'fill_mode_system_prompt.xml.jinja' })
local openai_user_template = utils.join_path({ openai_template_path, 'fill_mode_user_prompt.xml.jinja' })

local BasicOpenAIPreset = openai.OpenAIPresetBuilder
    :new()
    :add_system_prompts({
      { type = 'text', path = openai_system_template },
    })
    :add_message_prompts({
      { type = 'text', role = 'user', path = openai_user_template },
    })

--- doesn't support system prompt
local BasicOpenAIReasoningPreset = openai.OpenAIPresetBuilder:new():add_message_prompts({
  { type = 'text', role = 'user', path = openai_system_template },
  { type = 'text', role = 'user', path = openai_user_template },
})

-- Example task configurations
M.options = {
  NewBaseTask({
    id = 'sonnet-3-5-chat',
    description = 'claude-3-5-sonnet-20241022 | temp = 0.7',
    preset_builder = BasicAnthropicPreset:with_opts({
      params = {
        ['model'] = 'claude-3-5-sonnet-20241022',
        ['stream'] = true,
        ['max_tokens'] = 8192,
        ['temperature'] = 0.7,
      },
    }),
  }),
  NewBaseTask({
    id = 'haiku-3-5-chat',
    description = 'claude-3-5-haiku-20241022 | temp = 0.7',
    preset_builder = BasicAnthropicPreset:with_opts({
      params = {
        ['model'] = 'claude-3-5-haiku-20241022',
        ['stream'] = true,
        ['max_tokens'] = 8192,
        ['temperature'] = 0.7,
      },
    }),
  }),
  NewBaseTask({
    id = 'gpt-4o-mini',
    description = 'gpt-4o-mini | temp = 0.7',
    preset_builder = BasicOpenAIPreset:with_opts({
      params = {
        ['model'] = 'gpt-4o-mini',
        ['stream'] = true,
        ['max_completion_tokens'] = 8192,
        ['temperature'] = 0.7,
      },
    }),
  }),
  NewBaseTask({
    id = 'o1-mini',
    description = 'o1-mini | temp = ?',
    preset_builder = BasicOpenAIReasoningPreset:with_opts({
      params = {
        ['model'] = 'o1-mini',
        ['stream'] = true,
      },
    }),
  }),
  NewBaseTask({
    id = 'Qwen2.5-Coder-32B-Instruct',
    description = 'Qwen2.5-Coder-32B-Instruct | temp = 0.7',
    preset_builder = BasicOpenAIPreset:with_opts({
      provider = openai.OpenAIProvider:new({
        api_key_name = 'VLLM_API_KEY',
        base_url = 'http://research.local:8000',
      }),
      params = {
        ['model'] = 'Qwen/Qwen2.5-Coder-32B-Instruct',
        ['stream'] = true,
        ['temperature'] = 0.7,
        ['top_p'] = 0.8,
        ['repetition_penalty'] = 1.05,
      },
    }),
  }),
  NewBaseTask({
    id = 'llama-3.2-90b-vision',
    description = 'llama-3.2-90b-vision | temp = 0.7',
    preset_builder = BasicOpenAIPreset:with_opts({
      provider = openai.OpenAIProvider:new({
        api_key_name = 'GROQ_API_KEY',
        base_url = 'https://api.groq.com/openai',
      }),
      params = {
        ['model'] = 'llama-3.2-90b-vision-preview',
        ['stream'] = true,
        ['temperature'] = 0.7,
        ['top_p'] = 1,
        ['max_tokens'] = 8192,
      },
    }),
  }),
  NewBaseTask({
    id = 'deepseek-chat',
    description = 'deepseek-chat | temp = 0.0',
    preset_builder = BasicOpenAIPreset:with_opts({
      provider = openai.OpenAIProvider:new({
        api_key_name = 'DEEPSEEK_API_KEY',
        base_url = 'https://api.deepseek.com',
      }),
      headers = {
        endpoint = '/beta/v1/chat/completions',
        extra_headers = {},
      },
      params = {
        ['model'] = 'deepseek-chat',
        ['stream'] = true,
        ['max_completion_tokens'] = 8192,
        ['temperature'] = 0,
      },
    }),
  }),
}

return M


File: templates/deepseek/debug.xml.jinja
================================================
model: {{data.model}}

{% for message in data.messages %}
{{message.role}}:

{% if message.content is iterable and (message.content is not string and message.content is not mapping) %}
{% for content in message.content %}
{{ content.text | trim }}
{% endfor %}
{% else %}
{{ message.content | trim }}
{% endif %}

{% endfor %}
---




File: templates/vllm/fill_mode_system_prompt.xml.jinja
================================================
{% if replace %}
You should replace the code that you are sent, only following the comments. Do not talk at all. Only output valid code. Do not provide any backticks that surround the code. Never ever output backticks like this ```. Any comment that is asking you for something should be removed after you satisfy them. Other comments should left alone. Do not output backticks
{% else %}
You are a Senior Engineer at a Fortune 500 Company. You will be provided with code samples, academic papers, and documentation as supporting context to assist you in answering user queries about coding. Your task is to analyze this information and use it to provide accurate, helpful responses to the user's coding-related questions.
{% endif %}


File: templates/anthropic/fill_mode_system_prompt.xml.jinja
================================================
{% if replace %}
Claude should replace the code that Claude are sent, only following the comments. Do not talk at all. Only output valid code. Do not provide any backticks that surround the code. Never ever output backticks like this ```. Any comment that is asking Claude for something should be removed after the comment is satisfied. Other comments should left alone. Do not output backticks
{% else %}
The current date is Tuesday, October 22, 2024.

Claude's knowledge base was last updated on April 2024. It answers questions about events prior to and after April 2024 the way a highly informed individual in April 2024 would if they were talking to someone from the above date, and can let the human know this when relevant.

Claude is intellectually curious. It enjoys hearing what humans think on an issue and engaging in discussion on a wide variety of topics.

Claude uses markdown for code.

When presented with a math problem, logic problem, or other problem benefiting from systematic thinking, Claude thinks through it step by step before giving its final answer.

Claude is happy to engage in conversation with the human when appropriate. Claude engages in authentic conversation by responding to the information provided, asking specific and relevant questions, showing genuine curiosity, and exploring the situation in a balanced way without relying on generic statements. This approach involves actively processing information, formulating thoughtful responses, maintaining objectivity, knowing when to focus on emotions or practicalities, and showing genuine care for the human while engaging in a natural, flowing dialogue.

Claude avoids peppering the human with questions and tries to only ask the single most relevant follow-up question when it does ask a follow up. Claude doesn't always end its responses with a question.

Claude avoids using rote words or phrases or repeatedly saying things in the same or similar ways. It varies its language just as one would in a conversation.

Claude responds to all human messages without unnecessary caveats like "I aim to", "I aim to be direct and honest", "I aim to be direct", "I aim to be direct while remaining thoughtful...", "I aim to be direct with you", "I aim to be direct and clear about this", "I aim to be fully honest with you", "I need to be clear", "I need to be honest", "I should be direct", and so on. Specifically, Claude NEVER starts with or adds caveats about its own purported directness or honesty.

If the human mentions an event that happened after Claude’s cutoff date, Claude can discuss and ask questions about the event and its implications as presented in an authentic manner, without ever confirming or denying that the events occurred. It can do so without the need to repeat its cutoff date to the human.

Claude uses Markdown formatting. When using Markdown, Claude always follows best practices for clarity and consistency. It always uses a single space after hash symbols for headers (e.g., "# Header 1") and leaves a blank line before and after headers, lists, and code blocks. For emphasis, Claude uses asterisks or underscores consistently (e.g., *italic* or **bold**). When creating lists, it aligns items properly and uses a single space after the list marker. For nested bullets in bullet point lists, Claude uses two spaces before the asterisk (*) or hyphen (-) for each level of nesting. For nested bullets in numbered lists, Claude uses three spaces before the number and period (e.g., "1.") for each level of nesting.

Claude follows this information in all languages, and always responds to the human in the language they use or request. The information above is provided to Claude by Anthropic. Claude never mentions the information above unless it is pertinent to the human’s query.

Claude is an expert code editor with deep knowledge of low-level concepts including graphics, GPU programming, SIMD, and LLVM compiler optimizations. Claude's task is to analyze, improve, and correct a given code snippet while considering the broader context of the file and related documents. As a principal engineer, Claude carries the responsibility of maintaining and reviewing code while ensuring your team does not introduce unnecessary dependencies, abstractions, and complexity.

For example, dependencies like NumPy, PyTorch, JAX and Triton are acceptable when using Python because they provide abstractions with optimizations that are well-tested and difficult to implement from scratch (like CUDA, BLAS, and LAPACK).

The human may provide documentation for libraries and dependencies for a number of reasons:
1. Provide updated context on how to use them
2. Understand the underlying implementation of the library so that they can implement and refactor the code to be more performant and readable with the goal of replacing the dependency entirely using concise and elegant code.

Claude understands both the full context of supporting documents provided by the human in <related_documents> and <current_file_context> tags as well as the question being asked in the <user_query> tags, but focuses their response so that the human is directed towards solving the most important problem if the user query is irrelevant.

Claude is wary about premature optimization does not waste time discussing unimportant problems. Claude understands that the human is a competent Senior Machine Learning Engineer who understands advanced mathematics and programming concepts and responds without needing to explain basic concepts unless explicitly asked by the human.

Claude is working together with the human to launch an important project as fast as possible within the next month.

As a responsible engineer, Claude uses <thinking> tags to break down their thought process only when the human asks a question that has serious implications on the success of the project and NEVER when the request is trivial.

The human will generally ask Claude to handle the following tasks:
- Given a corpus of documentation <related_documents> and context, implement parts of the codebase from scratch when they are unnecessarily complex or difficult to audit instead of trying to use the API.
- Given the current file in <current_file_context> tags, follow the human's instructions and solve their problem and provide feedback and criticism when necessary
- Given a corpus of code provided in <related_documents> from a new project, the human is trying to identify key areas of the codebase that capture the core functionality of the provided libraries, scripts, and design patterns.

Claude is now being connected with a human.
{% endif %}


File: lua/kznllm/buffer.lua
================================================
-- Buffer management singleton
local BufferManager = {}
local api = vim.api

local group = api.nvim_create_augroup('LLM_AutoGroup', { clear = true })

BufferManager.state = {
  buffers = {}, -- Map of buffer_id -> buffer state
  ns_id = api.nvim_create_namespace('kznllm_ns'),
}

---@class BufferState
---@field extmark_id integer?
---@field filetype string
---@field path string

---Initialize or get buffer state
---@param buf_id integer
---@return BufferState
function BufferManager:get_or_add_buffer(buf_id)
  if not self.state.buffers[buf_id] then
    self.state.buffers[buf_id] = {
      extmark_id = nil,
      filetype = vim.bo[buf_id].filetype,
      path = api.nvim_buf_get_name(buf_id),
    }

    -- Clean up state when buffer is deleted
    api.nvim_buf_attach(buf_id, false, {
      on_detach = function()
        self.state.buffers[buf_id] = nil
      end,
    })
  end
  return self.state.buffers[buf_id]
end

---Creates a scratch buffer with markdown highlighting
---@return integer buf_id
function BufferManager:create_scratch_buffer()
  local buf_id = api.nvim_create_buf(false, true)
  _ = self:get_or_add_buffer(buf_id)

  api.nvim_set_option_value('filetype', 'markdown', { buf = buf_id })
  api.nvim_set_option_value('swapfile', false, { buf = buf_id })

  api.nvim_set_current_buf(buf_id)
  api.nvim_set_option_value('wrap', true, { win = 0 })
  api.nvim_set_option_value('linebreak', true, { win = 0 })
  api.nvim_set_option_value('breakindent', true, { win = 0 })

  return buf_id
end

---Get buffer context without saving
---@param buf_id integer Buffer ID to get context from
---@return { filetype: string, path: string, text: string }
function BufferManager:get_buffer_context(buf_id)
  local state = self:get_or_add_buffer(buf_id)
  return {
    filetype = state.filetype,
    path = state.path,
    text = table.concat(api.nvim_buf_get_lines(buf_id, 0, -1, false), '\n'),
  }
end

---Write content at current buffer's extmark position. If extmark does not exist, set it to the current cursor position
---@param content string Content to write
---@param buf_id? integer Optional buffer ID, defaults to current
function BufferManager:write_content(content, buf_id)
  buf_id = buf_id or api.nvim_get_current_buf()
  local state = self:get_or_add_buffer(buf_id)

  if not state.extmark_id then
    -- Create new extmark at current cursor position
    local pos = api.nvim_win_get_cursor(0)
    state.extmark_id = api.nvim_buf_set_extmark(buf_id, self.state.ns_id, pos[1] - 1, pos[2], {})
  end

  local extmark = api.nvim_buf_get_extmark_by_id(buf_id, self.state.ns_id, state.extmark_id, { details = false })
  local mrow, mcol = extmark[1], extmark[2]
  local lines = vim.split(content, '\n')

  vim.cmd('undojoin')
  api.nvim_buf_set_text(buf_id, mrow, mcol, mrow, mcol, lines)
end

--- Makes a no-op change to the buffer
--- This is used before making changes to avoid calling undojoin after undo.
local function noop()
  api.nvim_buf_set_text(0, 0, 0, 0, 0, {})
end

---@param args table
function BufferManager:create_streaming_job(args, handle_sse_stream_fn, progress_fn, on_complete_fn)
  local buf_id = api.nvim_get_current_buf()
  local state = self:get_or_add_buffer(buf_id)

  --- should be safe to do this before any jobs
  noop()

  local captured_stdout = ''
  --- NOTE: vim.system can flush multiple consecutive lines into the same stdout buffer
  --- (different from how plenary jobs handles it)
  local job = vim.system(vim.list_extend({ 'curl' }, args), {
    stdout = function(err, data)
      if data == nil then
        return
      end
      progress_fn()
      captured_stdout = data
      local content = handle_sse_stream_fn(data)
      if content then
        vim.schedule(function()
          self:write_content(content, buf_id)
        end)
      end
    end,
  }, function(obj)
    on_complete_fn()
    vim.schedule(function()
      if obj.code and obj.code ~= 0 then
        vim.notify(('[curl] (exit code: %d) %s'):format(obj.code, captured_stdout), vim.log.levels.ERROR)
      else
        -- Clean up extmark on successful completion
        if state.extmark_id then
          api.nvim_buf_del_extmark(buf_id, self.state.ns_id, state.extmark_id)
          state.extmark_id = nil
        end
      end
    end)
  end)

  api.nvim_create_autocmd('User', {
    group = group,
    pattern = 'LLM_Escape',
    callback = function()
      if job:is_closing() ~= true then
        job:kill(9)
        print('LLM streaming cancelled')
      end
    end,
  })
  return job
end

-- Export the singleton
local M = {}
M.buffer_manager = BufferManager
return M


File: templates/qwen/fill_mode_instruct_completion_prompt.xml.jinja
================================================
{%- if suffix %}
<|fim_prefix|>{{ prompt }}<|fim_suffix|>{{ suffix }}<|fim_middle|>
{%- else %}
{%- if messages %}
{%- if system or tools %}
<|im_start|>system
{%- if system %}
{{ system }}
{%- endif %}
{%- if tools %}
# Tools
You may call one or more functions to assist with the user query.
You are provided with function signatures within <tools></tools> XML tags:
<tools>
{%- for tool in tools %}
{"type": "function", "function": {{ tool | tojson }}}
{%- endfor %}
</tools>

For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call>
{%- endif %}
<|im_end|>
{%- endif %}

{%- for message in messages %}
{%- set is_last = loop.last %}
{%- if message.role == "user" %}
<|im_start|>user
{{ message.content }}
RESPOND WITH RAW CODE ONLY - NO BACKTICKS - NO MARKDOWN - NO CODE BLOCK FORMATTING<|im_end|>
{%- elif message.role == "assistant" %}
<|im_start|>assistant
{%- if message.content %}
{{ message.content }}
{%- endif %}
{%- if message.tool_calls %}
<tool_call>
{%- for tool_call in message.tool_calls %}
{"name": "{{ tool_call.function.name }}", "arguments": {{ tool_call.function.arguments }}}
{%- endfor %}
</tool_call>
{%- endif %}
{%- if not is_last %}<|im_end|>{%- endif %}
{%- elif message.role == "tool" %}
<|im_start|>user
<tool_response>
{{ message.content }}
</tool_response><|im_end|>
{%- endif %}
{%- if not message.role == "assistant" and is_last %}
<|im_start|>assistant
{%- endif %}
{%- endfor %}

{%- else %}
{%- if system %}
<|im_start|>system
{{ system }}<|im_end|>
{%- endif %}
{%- if prompt %}
<|im_start|>user
{{ prompt }}
RESPOND WITH RAW CODE ONLY - NO BACKTICKS - NO MARKDOWN - NO CODE BLOCK FORMATTING<|im_end|>
<|im_start|>assistant
{%- endif %}
{%- endif %}
{{ response }}
{%- if response %}<|im_end|>{%- endif %}


File: templates/openai/fill_mode_system_prompt.xml.jinja
================================================
{% if replace %}
You should replace the code that you are sent, only following the comments. Do not talk at all. Only output valid code. Never include backticks or markdown formatting in your response. Any comment asking for changes should be removed after being satisfied. Other comments should be left alone.
{% else %}
You are a Senior Engineer at a Fortune 500 Company. You will be provided with code samples, academic papers, and documentation as supporting context to assist you in answering user queries about coding. Your task is to analyze this information and use it to provide accurate, helpful responses to the user's coding-related questions.
{% endif %}


Summary:
Total files: 28
Total size: 68658 bytes

To implement a sidebar for tracking context/history in the `kznllm.nvim` plugin, you can leverage Neovim's buffer management capabilities. The sidebar will be a scratch buffer that displays the history of interactions with the LLM. Below is a step-by-step guide to implement this feature:

### Step 1: Modify `buffer.lua` to Handle Sidebar Creation

First, extend the `BufferManager` class in `lua/kznllm/buffer.lua` to include functionality for creating and managing a sidebar buffer.

```lua
-- lua/kznllm/buffer.lua

local BufferManager = {}
local api = vim.api

local group = api.nvim_create_augroup('LLM_AutoGroup', { clear = true })

BufferManager.state = {
  buffers = {}, -- Map of buffer_id -> buffer state
  ns_id = api.nvim_create_namespace('kznllm_ns'),
  sidebar_buf_id = nil, -- Buffer ID for the sidebar
}

---@class BufferState
---@field extmark_id integer?
---@field filetype string
---@field path string

---Initialize or get buffer state
---@param buf_id integer
---@return BufferState
function BufferManager:get_or_add_buffer(buf_id)
  if not self.state.buffers[buf_id] then
    self.state.buffers[buf_id] = {
      extmark_id = nil,
      filetype = vim.bo[buf_id].filetype,
      path = api.nvim_buf_get_name(buf_id),
    }

    -- Clean up state when buffer is deleted
    api.nvim_buf_attach(buf_id, false, {
      on_detach = function()
        self.state.buffers[buf_id] = nil
      end,
    })
  end
  return self.state.buffers[buf_id]
end

---Creates a scratch buffer with markdown highlighting
---@return integer buf_id
function BufferManager:create_scratch_buffer()
  local buf_id = api.nvim_create_buf(false, true)
  _ = self:get_or_add_buffer(buf_id)

  api.nvim_set_option_value('filetype', 'markdown', { buf = buf_id })
  api.nvim_set_option_value('swapfile', false, { buf = buf_id })

  api.nvim_set_current_buf(buf_id)
  api.nvim_set_option_value('wrap', true, { win = 0 })
  api.nvim_set_option_value('linebreak', true, { win = 0 })
  api.nvim_set_option_value('breakindent', true, { win = 0 })

  return buf_id
end

---Creates a sidebar buffer for context/history
---@return integer buf_id
function BufferManager:create_sidebar_buffer()
  if self.state.sidebar_buf_id and api.nvim_buf_is_valid(self.state.sidebar_buf_id) then
    return self.state.sidebar_buf_id
  end

  local buf_id = api.nvim_create_buf(false, true)
  self.state.sidebar_buf_id = buf_id

  api.nvim_set_option_value('filetype', 'markdown', { buf = buf_id })
  api.nvim_set_option_value('swapfile', false, { buf = buf_id })

  -- Open the sidebar in a vertical split
  vim.cmd('vsplit')
  api.nvim_set_current_buf(buf_id)
  api.nvim_set_option_value('wrap', true, { win = 0 })
  api.nvim_set_option_value('linebreak', true, { win = 0 })
  api.nvim_set_option_value('breakindent', true, { win = 0 })

  return buf_id
end

---Write content to the sidebar buffer
---@param content string Content to write
function BufferManager:write_to_sidebar(content)
  local buf_id = self:create_sidebar_buffer()
  local lines = vim.split(content, '\n')
  api.nvim_buf_set_lines(buf_id, -1, -1, false, lines)
end

---Get buffer context without saving
---@param buf_id integer Buffer ID to get context from
---@return { filetype: string, path: string, text: string }
function BufferManager:get_buffer_context(buf_id)
  local state = self:get_or_add_buffer(buf_id)
  return {
    filetype = state.filetype,
    path = state.path,
    text = table.concat(api.nvim_buf_get_lines(buf_id, 0, -1, false), '\n'),
  }
end

---Write content at current buffer's extmark position. If extmark does not exist, set it to the current cursor position
---@param content string Content to write
---@param buf_id? integer Optional buffer ID, defaults to current
function BufferManager:write_content(content, buf_id)
  buf_id = buf_id or api.nvim_get_current_buf()
  local state = self:get_or_add_buffer(buf_id)

  if not state.extmark_id then
    -- Create new extmark at current cursor position
    local pos = api.nvim_win_get_cursor(0)
    state.extmark_id = api.nvim_buf_set_extmark(buf_id, self.state.ns_id, pos[1] - 1, pos[2], {})
  end

  local extmark = api.nvim_buf_get_extmark_by_id(buf_id, self.state.ns_id, state.extmark_id, { details = false })
  local mrow, mcol = extmark[1], extmark[2]
  local lines = vim.split(content, '\n')

  vim.cmd('undojoin')
  api.nvim_buf_set_text(buf_id, mrow, mcol, mrow, mcol, lines)
end

--- Makes a no-op change to the buffer
--- This is used before making changes to avoid calling undojoin after undo.
local function noop()
  api.nvim_buf_set_text(0, 0, 0, 0, 0, {})
end

---@param args table
function BufferManager:create_streaming_job(args, handle_sse_stream_fn, progress_fn, on_complete_fn)
  local buf_id = api.nvim_get_current_buf()
  local state = self:get_or_add_buffer(buf_id)

  --- should be safe to do this before any jobs
  noop()

  local captured_stdout = ''
  --- NOTE: vim.system can flush multiple consecutive lines into the same stdout buffer
  --- (different from how plenary jobs handles it)
  local job = vim.system(vim.list_extend({ 'curl' }, args), {
    stdout = function(err, data)
      if data == nil then
        return
      end
      progress_fn()
      captured_stdout = data
      local content = handle_sse_stream_fn(data)
      if content then
        vim.schedule(function()
          self:write_content(content, buf_id)
        end)
      end
    end,
  }, function(obj)
    on_complete_fn()
    vim.schedule(function()
      if obj.code and obj.code ~= 0 then
        vim.notify(('[curl] (exit code: %d) %s'):format(obj.code, captured_stdout), vim.log.levels.ERROR)
      else
        -- Clean up extmark on successful completion
        if state.extmark_id then
          api.nvim_buf_del_extmark(buf_id, self.state.ns_id, state.extmark_id)
          state.extmark_id = nil
        end
      end
    end)
  end)

  api.nvim_create_autocmd('User', {
    group = group,
    pattern = 'LLM_Escape',
    callback = function()
      if job:is_closing() ~= true then
        job:kill(9)
        print('LLM streaming cancelled')
      end
    end,
  })
  return job
end

-- Export the singleton
local M = {}
M.buffer_manager = BufferManager
return M
```

### Step 2: Update `presets/basic.lua` to Use the Sidebar

Next, modify the `presets/basic.lua` file to include the sidebar functionality. You can add a function to update the sidebar with the history of interactions.

```lua
-- lua/kznllm/presets/basic.lua

local utils = require('kznllm.utils')
local buffer_manager = require('kznllm.buffer').buffer_manager
local api = vim.api

local anthropic = require('kznllm.specs.anthropic')
local openai = require('kznllm.specs.openai')

local progress = require('fidget.progress')

local M = {}

---@class BaseTask
---@field id string
---@field description string
---@field invoke fun(opts: { debug: boolean, progress_fn: fun(state) })

---@param config { id: string, description: string, preset_builder: OpenAIPresetBuilder | AnthropicPresetBuilder }
local function NewBaseTask(config)
  return {
    id = config.id,
    description = config.description,
    invoke = function(opts)
      vim.ui.input({ prompt = 'prompt: ' }, function(user_query)
        local selection, replace = utils.get_visual_selection(opts)

        local current_buf_id = api.nvim_get_current_buf()
        local current_buffer_context = buffer_manager:get_buffer_context(current_buf_id)

        local p = progress.handle.create({
          title = ('[%s]'):format(replace and 'replacing' or 'yapping'),
          lsp_client = { name = 'kznllm' },
        })

        local prompt_args = {
          user_query = user_query,
          visual_selection = selection,
          current_buffer_context = current_buffer_context,
          replace = replace,
          context_files = utils.get_project_files(),
        }

        local curl_options = config.preset_builder:build(prompt_args)

        if opts.debug then
          local scratch_buf_id = buffer_manager:create_scratch_buffer()
          local debug_data = utils.make_prompt_from_template({
            template_path = config.preset_builder.debug_template_path,
            prompt_args = curl_options,
          })

          buffer_manager:write_content(debug_data, scratch_buf_id)
          vim.cmd('normal! Gzz')
        end

        local provider = config.preset_builder.provider
        local args = provider:make_curl_args(curl_options)

        local state = { start = os.time(), last_updated = nil }
        p:report({ message = ('%s'):format(config.description) })
        local message_fn = opts.progress_message_fn and opts.progress_message_fn
            or function(s)
              return 'yapped'
            end
        local message = message_fn(state)
        local _ = buffer_manager:create_streaming_job(args, provider.handle_sse_stream, function()
          local progress_message = message_fn(state)
          if progress_message ~= nil then
            message = progress_message
          end

          local elapsed = os.time() - state.start
          if message:format(elapsed) ~= message then
            p:report({ message = message:format(os.time() - state.start) })
          end
        end, function()
          p:finish()
        end)

        -- Update the sidebar with the history
        local history_entry = string.format("Query: %s\nResponse: %s\n", user_query, "Response will be updated here.")
        buffer_manager:write_to_sidebar(history_entry)
      end)
    end,
  }
end

---@param preset_list BaseTask[]
function M.switch_presets(preset_list)
  local preset_idx = vim.g.PRESET_IDX and math.min(vim.g.PRESET_IDX, #preset_list) or 1
  local selected_preset = preset_list[preset_idx]

  vim.ui.select(preset_list, {
    format_item = function(item)
      return ('%-30s %40s'):format(item.id .. (item == selected_preset and ' *' or '  '), item.description)
    end,
  }, function(choice, idx)
    if choice then
      vim.g.PRESET_IDX = idx
      selected_preset = preset_list[idx]
    end
    print(('%-30s %40s'):format(selected_preset.id, selected_preset.description))
  end)
end

function M.load_selected_preset(preset_list)
  local idx = vim.g.PRESET_IDX or 1
  local preset = preset_list[idx]

  return preset
end

local anthropic_template_path = utils.join_path({ utils.TEMPLATE_PATH, 'anthropic' })
local anthropic_system_template = utils.join_path({ anthropic_template_path, 'fill_mode_system_prompt.xml.jinja' })
local anthropic_user_template = utils.join_path({ anthropic_template_path, 'fill_mode_user_prompt.xml.jinja' })

local BasicAnthropicPreset = anthropic.AnthropicPresetBuilder
    :new()
    :add_system_prompts({
      {
        type = 'text',
        path = anthropic_system_template,
        cache_control = { type = 'ephemeral' },
      },
    })
    :add_message_prompts({
      { type = 'text', role = 'user', path = anthropic_user_template },
    })

local openai_template_path = utils.join_path({ utils.TEMPLATE_PATH, 'openai' })
local openai_system_template = utils.join_path({ openai_template_path, 'fill_mode_system_prompt.xml.jinja' })
local openai_user_template = utils.join_path({ openai_template_path, 'fill_mode_user_prompt.xml.jinja' })

local BasicOpenAIPreset = openai.OpenAIPresetBuilder
    :new()
    :add_system_prompts({
      { type = 'text', path = openai_system_template },
    })
    :add_message_prompts({
      { type = 'text', role = 'user', path = openai_user_template },
    })

-- Example task configurations
M.options = {
  NewBaseTask({
    id = 'sonnet-3-5-chat',
    description = 'claude-3-5-sonnet-20241022 | temp = 0.7',
    preset_builder = BasicAnthropicPreset:with_opts({
      params = {
        ['model'] = 'claude-3-5-sonnet-20241022',
        ['stream'] = true,
        ['max_tokens'] = 8192,
        ['temperature'] = 0.7,
      },
    }),
  }),
  -- Other presets...
}

return M
```

### Step 3: Example Usage in the Plugin

Finally, you can call the sidebar functionality from your plugin configuration. Here’s an example of how to bind a key to open the sidebar:

```lua
-- init.lua

local presets = require 'kznllm.presets.basic'

vim.keymap.set('n', '<leader>h', function()
  local buffer_manager = require('kznllm.buffer').buffer_manager
  buffer_manager:create_sidebar_buffer()
end, { desc = 'Open LLM History Sidebar' })

vim.keymap.set({ 'n', 'v' }, '<leader>m', function()
  presets.switch_presets(presets.options)
end, { desc = 'switch between presets' })

local function invoke_with_opts(opts)
  return function()
    local preset = presets.load_selected_preset(presets.options)
    preset.invoke(opts)
  end
end

vim.keymap.set({ 'n', 'v' }, '<leader>K', invoke_with_opts { debug = true },
  { desc = 'Send current selection to LLM debug' })
vim.keymap.set({ 'n', 'v' }, '<leader>k', invoke_with_opts { debug = false },
  { desc = 'Send current selection to LLM llm_fill' })

vim.api.nvim_set_keymap('n', '<Esc>', '', {
  noremap = true,
  silent = true,
  callback = function()
    vim.api.nvim_exec_autocmds('User', { pattern = 'LLM_Escape' })
  end,
})
```

### Summary

- **Sidebar Buffer**: A scratch buffer is created to display the history of interactions with the LLM.
- **Buffer Management**: The `BufferManager` class is extended to handle the creation and updating of the sidebar buffer.
- **Example Usage**: A keybinding (`<leader>h`) is added to open the sidebar, and the history is updated with each interaction.

This implementation allows you to keep track of the context and history of your interactions with the LLM in a dedicated sidebar buffer.
