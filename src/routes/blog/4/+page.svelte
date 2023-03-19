
<script>
    import Blog from "./blog.md";
    let title = "Let's get started with Neovim"
    let date = "2023-02-01"
    let subtitle = "Setting up a basic Neovim config with lazy.nvim"
    let topics = ["Neovim", "Lua"]
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
