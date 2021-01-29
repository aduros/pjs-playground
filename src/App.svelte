<script>
    // let command = "pjs _.text --html b";
    // let input = `<foo id="aoeu"><b>Test</b></foo>`
    // let command = "pjs _.test --json .";
    // let input = `{"test":444}{"test":123}`;
    let command = "pjs _.age --csv-header";
    let input = `name,age
bruno,33
abc,11
xyz,50
`;

    let lastError = null;
    let generatedProgram = null;
    let programOutput = null;

    import { PassThrough } from "stream";
    import shellParser from "shell-parser";

    import cli from "pjs-tool/lib/cli";
    import generate from "pjs-tool/lib/generate";
    import csvIterator from "pjs-tool/lib/csv-iterator";
    import jsonIterator from "pjs-tool/lib/json-iterator";
    import htmlIterator from "pjs-tool/lib/html-iterator";
    import xmlIterator from "pjs-tool/lib/xml-iterator";

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
        try {
            const args = shellParser(command);
            if (args[0] != "pjs") {
                throw new Error("Command must start with pjs");
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
                            resolve(fn(process, null, dummyRequire).catch(function (error) {
                                console.error(error);
                                throw new Error("An error occurred when running your script. Consider using --explain to help debug.\n\n"+error.message);
                            }));
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
            programOutput = error.message;
            generatedProgram = null;

            console.error(error);
        }
    }

    $: run(command, input);
</script>

<style>
.hasError {
    color: red;
}
input, textarea {
    font-family: monospace;
}

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
}

.pjs-output {
    border: none;
}
textarea {
    resize: none;
}
</style>

<main class="vbox">
    <div class="vbox">
        <input bind:value={command} class="pjs-program {lastError ? "hasError" : ""}" spellcheck="false">
    </div>
    <div class="vbox flex1">
        <textarea bind:value={input} class="pjs-input flex1" spellcheck="false"></textarea>
        <textarea class="pjs-output flex1 {lastError ? "hasError" : ""}" readonly>{programOutput}</textarea>
    </div>
</main>
