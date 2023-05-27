<script>
    export let name;
    export let link;
    export let date;
</script>

<a href={link}>
    <article class=card>
        <h3>{name}</h3>
        <h3>{date}</h3>
    </article>
</a>

<style>
    a {
        margin: 5px auto;
        max-width: 500px; 
    }
    
    .card {
        text-align: center;
        box-shadow: rgba(0, 0, 0, 0.1) 0px 2px 4px;
        transition: all .2s ease-in-out;
        margin: 5px auto;
        border-radius: 15px;
        max-width: 500px;
        width: 350px;
        height: 200px;
    }

    h3 {
        font-size: 20px;
    }

    .card:hover {
        box-shadow: rgba(101, 4, 204, 0.3) 0px 5px 10px;
        cursor: pointer;
        transform: translate3d(0px, -1.25px, 0px);
    }

    a:hover, a:focus {
        text-decoration: none;
        background-color: none;
        --background-color:none;
    }
</style>
