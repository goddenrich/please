go_library(
    name = "plzinit",
    srcs = glob(
        ["*.go"],
        exclude = ["*_test.go"],
    ),
    visibility = ["PUBLIC"],
    deps = [
        "//src/assets",
        "//src/cli",
        "//src/core",
        "//src/fs",
        "//src/scm",
        "//third_party/go:levenshtein",
        "//third_party/go:logging",
    ],
)

go_test(
    name = "plzinit_test",
    srcs = ["init_test.go"],
    deps = [
        ":plzinit",
        "//src/fs",
        "//third_party/go:testify",
    ],
)
