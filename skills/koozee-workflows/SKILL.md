---
name: koozee-workflows
description: Use when the user wants Koozee image, video, text, Canvas, Koozee Skill, workbench, credits, upload, or canvas editing through the Koozee MCP.
---

# Koozee workflows

Use the `koozee` MCP server. Do not invent IDs, URLs, media bytes, slot names, or option values. Read the tool response or catalog first. A call without login returns 401. Let the host finish OAuth. Do not send a Bearer token or client secret.

## Pick a tool

- Account uid: `get_profile`.
- Balance or a recharge link: `credits`. Report the number and give `rechargeUrl`. Open that URL when the user wants to pay. Do not open `/pricing` without the returned hash, and do not add a language prefix. Do not call `estimate` with an empty `tasks` to read the balance.
- Cost of a planned batch: one `estimate` whose `tasks[]` includes every `generate`, `skill_run`, and `workbench_run`. If `needsRecharge`, open the returned `rechargeUrl`, then `estimate` again. Do not submit while credits are short. Skip `estimate` only when the user already chose not to be reminded and the balance is enough.
- Reminder preference: `prefs`. "Remind me every time" sets `creditsRemind` to `always`. "Don't remind me" sets `dont_remind`. This is not `get_profile` and not `estimate`.
- Free image, video, or text: `list_models`, then `generate`. Copy `options`, `caps`, and whether `prompt` is required from that catalog entry. Try-on, flat lay, pose change, print swap, wrinkle removal, model swap, and clothing recolor are not `generate`. E-commerce detail pages are `generate`.
- Canvas products and published Koozee Skills: `list_skills`, then `get_skill`, then `skill_run`. Unpublished Koozee Skills are out of scope. Workbench-only pages are not `skill_run`.
- Pages that are not on the canvas (quality repair, video upscale, AI erase, sketch, OOTD): `list_workbench_tools`, then `workbench_run`.
- A local image, video, audio, or text file: `upload` before any runner. A host that can PUT sends the filename, PUTs `uploadUrl` with the returned `mime`, and then uses `url`. ChatGPT cannot PUT: send base64 `content` and use the returned `url` and `kind`. A file dropped into the chat is not uploaded. With no local path, ask for a path or a public URL. Do not call `upload` with empty bytes. A text file has `kind` `doc`; do not treat it as inline text.
- Unknown node id: `project_get`.
- Delete one workbench History row: `history_delete`.
- Soft-delete a workflow project: `project_delete`.

## Surface

Omitting `surface` on `generate`, `skill_run`, `await`, and `write` means workflow. A workbench-only page uses `workbench_run`; omitting `surface` there still means workbench. Use `skill_run` with `surface=workbench` only when the user asks for the workbench. `link`, `project_get`, and `project_delete` accept only `workflow`. `history_delete` accepts only `workbench`. `write` may set `surface=workbench`, and then it does not change the project.

## Workflow project

Nodes in one conversation belong to one project. If the user gives `/app/canvas/{id}`, use that id. Otherwise the first `write` creates the project. Later calls pass the returned `projectId`. Omitting it still appends while that project is active. A successful `await` includes `projectId` when one exists. `project_delete` does not keep that sticky project.

After the canvas changes, give the user the absolute `url` from `write`, `link`, or `project_get` and tell them to open it. Paste that link once at the end. Do not use the API host as the page URL.

## Runs

Image and video `generate` return `taskId` immediately. Text `generate` returns `text` immediately. Do not `await` text.

`skill_run` and `workbench_run` return `taskId`. A batch also returns `taskIds`, and `taskId` is the first item. `await` each id. `await` reads History once for that `taskId` and `surface`. Flattened rows arrive in `results[]`. If any row is pending, `results` is empty; call `await` again.

Call `estimate` before submitting a batch.

### Free image or video

1. `list_models`. Use that entry's options. Do not invent values.
2. `generate` with `model`. For video, `duration` is that entry's integer seconds (omit for 5). `generateAudio` omits to audio on; `false` turns it off. Put public reference URLs in `images`, `videos`, or `audios`.
3. On workflow, `write` at once with that `taskId` and no `url`. Video also needs `kind=video`. The card stays pending. Skip this on the workbench.
4. When this step merges cards, `link` each source to that pending `nodeId`, then `await`.
5. `await` the same surface. Call again while pending.
6. On success, `write` that `nodeId` and `results[].url`, plus the `model` and `aspectRatio` just used. Video also sends `kind=video` and the same `duration` and `generateAudio`. Omitted `kind` is an image.
7. On failure, `write` that `nodeId` with `failed: true` and no `url`.

### Text

1. `list_models`, then `generate` with `model` and `prompt`.
2. Read `text`. Do not `await`.
3. To place it on the canvas, `write` with `kind=text` and that full text.

Adding a prompt is `write` with `kind=text`. Do not `generate`, and do not omit `url` to drop an empty image card. An empty card is only when the user asks to place a card and not generate yet.

### Canvas product or Koozee Skill

1. `get_skill` for the product id or Koozee Skill id.
2. On workflow, `write` the task card first: `kind=product` or `kind=skill`. Put option slots in `values`, using the same keys as the later `skill_run`. Skip the card only when the user wants a URL only, does not want the canvas, or asked for the workbench.
3. `write` reference cards when needed, then `link` with `from`, `to`, and `slot`. `slot` matches a `get_skill` input key. If the source card is already in the project, only `link` it. Media slots must be linked. Do not leave the canvas slot empty by putting the URL only in `skill_run`.
4. `skill_run` with `skill` and `inputs`. Do not build the body from edges. Upload each local file, then put the URLs in one slot array. A media slot takes one URL, or up to `maxConnections`. A text slot takes a string.
5. `await` each task id on the same surface.
6. On success, `write` once per result URL with the task card `nodeId`. Do not use the `skill_run` task id to create a pending image card. Do not `link` the results.
7. On failure, `write` the task card `nodeId` with `failed: true` and no `url`.

Do not create a pending card from a `skill_run` or `workbench_run` id. Pending cards are for a `generate` task id.

### Workbench-only page

1. `list_workbench_tools`.
2. `workbench_run` with `tool` and `inputs`.
3. `await` with `surface=workbench`.
4. Do not `write`. The result is already in History. If you still `write`, `surface` must stay `workbench`.

## Links and cards

`link` `from` and `to` are node ids, not URLs. A link onto a task card requires `slot`.

Merge, chain, add, or combine cards by `link` from each source to the result. Sources may be text, image, or video. Do not only paste them into a `generate` prompt or media array. A single card that was not asked to be merged does not need a link. Do not `link` the fan-out results of a product or Koozee Skill.

An empty image, video, or text card is `write` with `url` or `text` omitted, plus the prompt and options. It cannot include `nodeId`. Omitted `model` uses the canvas default.

To refill an existing card, pass `projectId` and the previous `nodeId`. Refill does not rename the card.

To clear one workbench record, `history_delete` with the last `await` `taskId`. To discard a canvas, `project_delete` with the last `write` `projectId`.

## Out of scope

Unpublished Koozee Skills. Submitting a run from incoming edges. Uploading a chat attachment that has no local path. Listing History. Restoring the trash. Deleting a canvas node. Talking-head video, poster design, and fashion video.
