<script>
    import { PassThrough } from "stream";
    import shellParser from "shell-parser";

    import Textfield from '@smui/textfield';
    import HelperText from '@smui/textfield/helper-text/index';
    import Tab, {Label as TabLabel, Icon as TabIcon} from '@smui/tab';
    import TabBar from '@smui/tab-bar';
    import Button, {Label as ButtonLabel, Icon} from '@smui/button';
    import List, {Item, Separator, Text, PrimaryText, SecondaryText, Graphic} from '@smui/list';
    import Menu from '@smui/menu';

    import cli from "pjs-tool/lib/cli";
    import generate from "pjs-tool/lib/generate";
    import csvIterator from "pjs-tool/lib/csv-iterator";
    import jsonIterator from "pjs-tool/lib/json-iterator";
    import htmlIterator from "pjs-tool/lib/html-iterator";
    import xmlIterator from "pjs-tool/lib/xml-iterator";

    let command = "pjs '_.toUpperCase()'";
    let input = [
        "There once was a man from Nantucket",
        "Who kept all his cash in a bucket.",
        "But his daughter named Nan,",
        "Ran away with a man",
        "And as for the bucket, Nantucket.",
    ].join("\n");

    let lastError = null;
    let generatedProgram = null;
    let programOutput = null;
    let outputTab;
    let copyMenu;

    if (location.hash) {
        try {
            let hash = JSON.parse(decodeURIComponent(location.hash).substr(1));
            command = hash.command;
            input = hash.input;
        } catch (error) {
        }
    }
    function setURLHash (command, input) {
        const hash = encodeURIComponent(JSON.stringify({
            command, input,
        }));
        history.replaceState("", "", "#"+hash);
    }
    $: setURLHash(command, input);

    function copyToClipboard (text) {
        return navigator.clipboard.writeText(text);
    }

    function streamToIterable (stream) {
        // TODO(2021-01-28): Make this truly async instead of loading the entire stream first
        const promise = new Promise((resolve, reject) => {
            let arr = [];
            stream.on("data", data => {
                arr.push(data);
            });
            stream.on("end", () => {
                resolve(arr);
            });
            stream.on("error", reject);
        });

        return {
            [Symbol.asyncIterator]: () => {
                let n = 0;
                return {
                    async next () {
                        const elements = await promise;
                        return n >= elements.length ? { done: true } : { value: elements[n++] };
                    }
                };
            }
        }
    }

    async function run (command, input) {
        let showSource = false;
        try {
            const args = shellParser(command);
            if (args[0] != "pjs") {
                throw new Error("Command must start with \"pjs\"");
            }

            let action;
            let output = "";

            function log (text) {
                output += text;
            }

            const dummyPjsModule = {
                print (...messages) {
                    messages = messages.map(o => (typeof o == "object") ? JSON.stringify(o, null, "  ") : o);
                    output += messages.join(" ")+"\n";
                },
                eachLine (inputStream) {
                    return input.split("\n");
                },
                eachCsv (inputStream, opts) {
                    return streamToIterable(csvIterator(inputStream, opts));
                },
                eachJson (inputStream, opts) {
                    return streamToIterable(jsonIterator(inputStream, opts));
                },
                eachHtml: htmlIterator,
                eachXml: xmlIterator,
            };
            function dummyRequire (name) {
                if (name == "pjs-tool") {
                    return dummyPjsModule;
                }
                throw new Error("Module not supported in playground: "+name);
            }

            const program = cli()
                .exitOverride()
                .configureOutput({
                    writeOut: log,
                    writeErr: log,
                    getOutHelpWidth: () => 80,
                    getErrHelpWidth: () => 80,
                })
                .action((script, files, options) => {
                    action = new Promise((resolve, reject) => {
                        // Must include at least one script
                        if (options.before == null && script == null && options.after == null) {
                            log(program.helpInformation());
                            resolve();
                            return;
                        }

                        let mode, markupSelector;
                        if (options.csv || options.csvHeader) {
                            mode = "csv";
                        } else if (options.json != null) {
                            mode = "json";
                        } else if (options.html != null) {
                            mode = "html";
                            markupSelector = options.html;
                        } else if (options.xml != null) {
                            mode = "xml";
                            markupSelector = options.xml;
                        }

                        generatedProgram = generate({
                            mode,
                            beforeJs: options.before ? options.before.join("\n;") : null,
                            lineJs: script,
                            afterJs: options.after ? options.after.join("\n;") : null,
                            inputStream: null,

                            textDelimiter: (options.delimiter != null) ? new RegExp(options.delimiter) : null,
                            csvHeader: options.csvHeader,
                            jsonFilter: options.json,
                            markupSelector: markupSelector,
                        });
                        showSource = true;

                        if (options.explain) {
                            log(generatedProgram+"\n");
                            resolve();

                        } else {
                            const duplex = new PassThrough();
                            duplex.end(input);

                            const process = {
                                stdin: duplex,
                            };

                            const fn = new Function("process", "FILENAME", "require",
                                "return " + generatedProgram);
                            resolve(fn(process, null, dummyRequire));
                        }
                    });
                });

            try {
                program.parse(["node"].concat(args));
            } catch (error) {
                // Suppress an error from Commander about missing Error.captureStackTrace
                // console.error(error);
                throw new Error(output);
            }

            await action;

            lastError = null;
            programOutput = output;

        } catch (error) {
            lastError = error;
            programOutput = "";
            if (!showSource) {
                generatedProgram = null;
            }

            console.error(error);
        }
    }
    $: run(command, input);

</script>

<style>
.hbox {
    display: flex;
}
.vbox {
    display: flex;
    flex-direction: column;
}
.flex1 {
    flex: 1;
}
main {
    height: 100%;
    max-width: 1140px;
    margin: auto;
}
textarea {
    resize: none;
    border: none;
    outline: none;
}
.spacer {
    margin-bottom: 10px;
}
</style>

<main class="vbox">
    <div style="margin: 0px 20px 0 20px">
      <h2>pjs playground</h2>
      <p><a href="https://github.com/aduros/pjs"><b>pjs</b></a> is a command-line tool for text processing.
      Try the interactive demo below or check out some <a href="https://github.com/aduros/pjs#examples">examples</a>.</p>
    </div>
    <div class="hbox" style="margin: 80px 20px 0 20px">
        <div class="flex1">
            <Textfield bind:value={command} style="width: 100%"></Textfield>
            <HelperText persistent style="color: red">{lastError ? lastError.message : ""}</HelperText>
        </div>
        <div style="margin-top: 10px">
            <Button on:click={() => copyMenu.setOpen(true)}>
                <Icon class="material-icons">content_copy</Icon>
                <ButtonLabel>Copy</ButtonLabel>
            </Button>
            <Menu bind:this={copyMenu}>
                <List>
                    <Item on:click={() => copyToClipboard(command)}><Graphic class="material-icons">code</Graphic> <Text>Copy pjs command</Text></Item>
                    <Separator />
                    <Item on:click={() => copyToClipboard(location.href)}><Graphic class="material-icons">link</Graphic> <Text>Copy playground URL</Text></Item>
                    <Item on:click={() => copyToClipboard(`[\`${command}\`](${location.href})`)}><Graphic class="material-icons">link</Graphic> <Text>Copy playground markdown</Text></Item>
                </List>
            </Menu>
        </div>
    </div>
    <div class="hbox flex1" style="margin: 0 20px 20px 20px">
        <div class="vbox flex1">
            <div class="spacer">
                <TabBar tabs={["Input"]} let:tab>
                    <Tab {tab} minWidth>
                        <TabLabel>{tab}</TabLabel>
                    </Tab>
                </TabBar>
            </div>
            <textarea bind:value={input} class="pjs-input flex1" spellcheck="false"></textarea>
        </div>
        <div class="vbox flex1">
            <div class="spacer">
                <TabBar tabs={["output", "source"]} let:tab bind:active={outputTab}>
                    <Tab {tab} minWidth>
                        <TabLabel>{tab}</TabLabel>
                    </Tab>
                </TabBar>
            </div>
            <textarea class="flex1" readonly>{outputTab == "output" ? programOutput : generatedProgram}</textarea>
        </div>
    </div>
</main>
