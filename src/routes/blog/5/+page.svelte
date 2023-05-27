
<script>
    import Blog from "./blog.md";
    let title = "Spring Cleaning with Rust"
    let date = "2023-05-21"
    let subtitle = "Building a Filewatcher"
    let topics = ["Rust", "TDD","CLI"]
</script>

<p class="date">{date}</p>
<h1>{title}</h1>
<h2>{subtitle}</h2>
<h2>{topics.join(" • ")}</h2>
<hr>
<Blog></Blog>

<style>
    h1, h2 {
        margin-top: 0px;
        text-align: center;
        padding-bottom: 20px;
    }
    p {
        padding-top: 20px;
    }

    .date {
        color:grey;
    }

    hr {
        padding-bottom: 1px;
    }

</style>
