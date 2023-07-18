<script lang="ts">
    import { Textarea, Label } from "flowbite-svelte";
    import { Button } from "flowbite-svelte";
    import {
        Table,
        TableBody,
        TableBodyCell,
        TableBodyRow,
        TableHead,
        TableHeadCell,
    } from "flowbite-svelte";
    $: text = "";
    $: list = [];
    $: result = [];
    $: notFound = [];
    async function convertTextToList() {
        const originalText = document.getElementById("original-id");
        if (originalText == null) {
            return;
        }
        text = originalText.value;
        if (text == null || text == "") {
            return;
        }
        const pyodide = window.pyodide;
        await pyodide.runPython(`
text = """${text}"""
seg_list = jieba.cut(text, cut_all=False)
seg_list = list(seg_list)
`);
        const segList = pyodide.globals.get("seg_list").toJs();
        const chineseOnly = segList.filter((word) => {
            const chineseRegex = /^[\u4e00-\u9fa5]+$/;
            return chineseRegex.test(word);
        });
        list = chineseOnly;
        for (let i = 0; i < list.length; i++) {
            const word = list[i];
            // console.log(word);
            await pyodide.runPython(
`word_entry = d.lookup('` + word.trim() + `')
not_found = []
data = []
if word_entry is None:
    pin = pinyin_jyutping_sentence.pinyin("` + word.trim() + `")
    pin = colorize_pinyin.colorized_HTML_string_from_string(pin)
    not_found.append({
        "simplified": "` + word.trim() + `",
        "traditional": "` + word.trim() + `",
        "pinyin": pin,
        "definitions": ""
    })
else:
    ch_sim = '<span class="ch-sim">' + word_entry.simp + '</span>'
    ch_trad = ''
    if word_entry.simp != word_entry.trad:
        ch_trad = '<span class="ch-trad">〔' + word_entry.trad  + '〕</span>'
    for entry in word_entry.definition_entries:
        pinyin = entry.pinyin
        tone = ptc().convert_text(pinyin)
        colored_pinyin = colorize_pinyin.colorized_HTML_string_from_string(tone)
        if not colored_pinyin:
            colored_pinyin = '<span class="pinYinWrapper"><span class="t5">' + tone + '</span></span>'
        data.append({
            "simplified": word_entry.simp,
            "traditional": word_entry.trad,
            "pinyin": colored_pinyin,
            "definitions": entry.definitions
        })
`
            );
            const chData = pyodide.globals.get("data").toJs();
            const chNotFound = pyodide.globals.get("not_found").toJs();
            if (chData != null && chData.length > 0) {
                result = result.concat(Object.fromEntries(chData[0]));
            }

            if (chNotFound != null && chNotFound.length > 0) {
                notFound = notFound.concat(Object.fromEntries(chNotFound[0]));
                const meaning = await getMeaning(notFound[0]);
                result = result.concat(meaning);
            }
        }
    }

    function getMeaning(word) {
        return new Promise(function (resolve, reject) {
            const translateUrl = `https://translate.googleapis.com/translate_a/single?client=gtx&sl=zh-CN&tl=en-US&dt=t&q=${word.simplified}`;
                const xhr = new XMLHttpRequest();
                xhr.open('GET', translateUrl, true);
                xhr.onreadystatechange = function () {
                    if (xhr.readyState === 4) {
                        if (xhr.status === 200) {
                            let responseText = xhr.responseText;
                            let res = JSON.parse(responseText);
                            const data = {
                                traditional: word.simplified,
                                simplified: word.traditional,
                                pinyin: word.pinyin,
                                definitions: [res[0][0][0]]
                            }
                            resolve([data]);
                        }
                    }
                }
                xhr.send();
        });
    }
</script>

<Label for="original-id" class="mb-2">Chinese Text</Label>
<Textarea
    id="original-id"
    placeholder="Enter Chinese Text Here..."
    rows="4"
    name="text"
/>

<br />
<div style="text-align: center;">
    <Button color="alternative" on:click={convertTextToList}>Convert</Button>
</div>
<br />

{#if result.length > 0}
    <Table shadow>
        <TableHead>
            <TableHeadCell>Simplified</TableHeadCell>
            <TableHeadCell>Traditional</TableHeadCell>
            <TableHeadCell>Pinyin</TableHeadCell>
            <TableHeadCell>Definitions</TableHeadCell>
        </TableHead>
        {#each result as item}
            {#if item != ""}
                <TableBody>
                    <TableBodyRow>
                        <TableBodyCell>{item.simplified}</TableBodyCell>
                        <TableBodyCell>{item.traditional}</TableBodyCell>
                        <TableBodyCell>{@html item.pinyin}</TableBodyCell>
                        <TableBodyCell>{item.definitions}</TableBodyCell>
                    </TableBodyRow>
                </TableBody>
            {/if}
        {/each}
    </Table>
{/if}

<style>
</style>
