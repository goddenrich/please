subinclude("//docs/build_defs")

deps = []

for src in glob(
    [
        "*.html",
        "milestones/*.html",
    ],
    exclude = [
        "template.html",
        "lexicon*.html",
        "language.html",
        "config.html",
        "error.html",
    ],
):
    deps += [html_template(name = src.replace(".", "_").replace("/", "_"), src = src)]

html_template(
    name = "codelabs_html",
    src = "//docs/codelabs:templated_index",
    output = "codelabs.html",
)

genrule(
    name = "rules",
    outs = ["rules.json"],
    cmd = '"$TOOL" query rules > "$OUT"',
    local = True,
    sandbox = False,
    tools = ["//src:please"],
)

genrule(
    name = "lexicon_html",
    srcs = deps + [
        "lexicon.html",
        "lexicon_entry.html",
        "template.html",
        ":rules",
    ],
    outs = ["lexicon.html"],
    cmd = [
        '"$TOOLS_LEX" > lexicon.html',
        '"$TOOLS_TMPL" --template docs/template.html --in lexicon.html > tmp.html',
        'mv tmp.html "$OUT"',
    ],
    tools = {
        "lex": ["//docs/tools/lexicon_templater"],
        "tmpl": ["//docs/tools/templater"],
    },
)

genrule(
    name = "config_html",
    srcs = [
        "config.html",
        "template.html",
    ],
    outs = ["config.html"],
    cmd = [
        '"$TOOLS_CONFIG" > config.html',
        '"$TOOLS_TMPL" --template docs/template.html --in config.html > tmp.html',
        'mv tmp.html "$OUT"',
    ],
    tools = {
        "config": ["//docs/tools/config_templater"],
        "tmpl": ["//docs/tools/templater"],
    },
)

genrule(
    name = "language_html",
    srcs = {
        "template": ["template.html"],
        "html": ["language.html"],
        "grammar": ["grammar.txt"],
    },
    outs = ["language.html"],
    cmd = '"$TOOLS_TEMPLATE" --template $SRCS_TEMPLATE --in $SRCS_HTML | "$TOOLS_GRAMMAR" > "$OUT"',
    tools = {
        "grammar": ["//docs/tools/grammar_templater"],
        "template": ["//docs/tools/templater"],
    },
)

filegroup(
    name = "docs",
    srcs = glob(["images/*.png"]) + deps + [
        ":language_html",
        ":lexicon_html",
        ":config_html",
        "tachyons-4.12.0.min.css",
        "styles.css",
        "action.js",
        "tabs.js",
        "error.html",
        "images/thoughtmachine.png",
        "plz-recording.json",
        "//third_party/js:asciinema",
        "favicon",
        ":codelabs_html",
    ],
    visibility = ["//docs/..."],
    deps = ["//docs/codelabs"],
)

#TODO(jpoole): unify these two once we switch to the new website
tarball(
    name = "tarball",
    srcs = [":docs"],
    out = "docs.tar.gz",
    labels = ["hlink:plz-out/pkg"],
)

# This is used exclusively for the s3 website
tarball(
    name = "deep-tarball",
    srcs = [
        ":docs",
        "//docs/codelabs",
    ],
    out = "deep-docs.tar.gz",
    flatten = False,
    labels = ["hlink:plz-out/pkg"],
)

filegroup(
    name = "template_html",
    srcs = ["template.html"],
    visibility = ["//docs/..."],
)

genrule(
    name = "performance_data",
    outs = ["performance/all_results.jsonl"],
    cmd = "curl https://get.please.build/performance/all_results.jsonl > $OUT",
)

# Convenient for popping open the docs in a browser after building (must use 'plz run').
sh_cmd(
    name = "view",
    cmd = "sensible-browser http://localhost:8080 && cd plz-out/gen/docs && python3 -m http.server 8080",
    data = [
        ":docs",
        ":performance_data",
    ],
)

sh_cmd(
    name = "server",
    cmd = "cd plz-out/gen/docs && python3 -m http.server 8080",
    data = [
        ":docs",
        ":performance_data",
    ],
)
