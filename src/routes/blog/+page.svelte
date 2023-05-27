<script>    

    import ArticleCard from "$lib/ArticleCard.svelte";
    const blogs = [
        ["Spring Cleaning with Rust", "2023-05-26", 5],
        ["Down the Neovim Rabbit Hole", "2023-03-19", 4],
        ["Let's write a blog in Rust - Part 3", "2023-03-12", 3],
        ["Let's write a blog in Rust - Part 2", "2023-02-02", 2],
        ["Let's write a blog in Rust - Part 1", "2023-01-27", 1],
    ]
</script>

<h1 class=title>Blogs</h1>
<div class=article-grid> 
    {#each blogs as blog}
         <ArticleCard name={blog[0]} date={blog[1]} link={`/blog/${blog[2]}`}></ArticleCard>
    {/each}
</div>

<style>

    .title {
        text-align: center;
    }
    .article-grid {
        justify-content: center;
        align-items: center;
        display: grid;
        grid-template-columns: auto auto auto;
        gap: 10px;
    }

    @media (min-width:320px){
      .article-grid {
        grid-template-columns: 1fr;
      }
    }
    @media (min-width:481px){
      .article-grid {
        grid-template-columns: 1fr;
      }
    }
    @media (min-width:641px) {
      .article-grid {
        grid-template-columns: 1fr;
      }
    }
    @media (min-width:961px) {
      .article-grid {
        grid-template-columns: 1fr;
      }
    }
    @media (min-width:1025px) {
      .article-grid {
        grid-template-columns: 1fr;
      }
    }
    @media (min-width:1281px) {
      .article-grid {
        grid-template-columns: auto auto auto;
      }
    }
</style>
