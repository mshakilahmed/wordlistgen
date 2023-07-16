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
            console.log(word);
            await pyodide.runPython(
`word_entry = d.lookup('` + word.trim() + `')
if word_entry is None:
	word_entry['trad'] = '` + word.trim() + `'
	word_entry['definition_entries'] = [{'pinyin': '', 'definitions': ['']}]

ch_sim = '<span class="ch-sim">' + word_entry.simp + '</span>'
ch_trad = ''
if word_entry.simp != word_entry.trad:
  ch_trad = '<span class="ch-trad">〔' + word_entry.trad  + '〕</span>'
data = []
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
print(data)
`
            );
            const chData = pyodide.globals.get("data").toJs();
            console.log(chData[0]);
            result = result.concat(Object.fromEntries(chData[0]));
        }
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
