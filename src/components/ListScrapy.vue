<template lang="pug">
  div
    //- span(v-html="iframe")
    span secret code: {{ secretCode }}
    ul
      li(v-for="link in links")
        a(:href="link" target="_blank") {{ link }}
    pre {{ $data }}
</template>

<script>
import cheerio from "cheerio";
import axios from "axios";

export default {
  data() {
    return {
      baseurl: "https://www.rexpelis.com/serie/12-monos",
      baseurlEpisodes: "https://www.rexpelis.com/player/embed/episode/",
      html: null,
      episodesCodes: [],

      secretCode: null,

      iframe: null,

      page: null,
      response: null,
      links: [],
      error: null
    };
  },

  mounted() {
    this.getWebSite();
  },

  watch: {
    response() {
      this.scrapy(this.html);
    },

    episodesCodes() {
      const embedUrl = `${this.baseurlEpisodes}${this.episodesCodes[0].playerId}`;
      console.log(embedUrl);

      axios
        .get(embedUrl)
        .then(res => {
          this.iframe = res.data;

          const $ = cheerio.load(this.iframe);

          const source = $(this.iframe).attr("src");
          const arrSecretCode = source.split("/");
          console.log(arrSecretCode);
          // this.secretCode = arrSecretCode[arrSecretCode.length];

          // console.log(this.secretCode);
        })
        .catch(err => (this.error = err));
    }
  },

  methods: {
    getWebSite() {
      axios
        .get(this.baseurl, {
          method: "get",
          redirect: "follow"
        })
        .then(res => {
          this.response = res;
          this.html = res.data;
        })
        .catch(err => (this.error = err));
    },

    scrapy(pHtml) {
      let $ = cheerio.load(pHtml);

      // const itemSeasons = $(".pelicula-player .item-season");

      const episodes = $(".item-season-episodes");

      //
      // Primera temporada
      const links = $(episodes[2]).find("a");
      const links_length = $(links).length;

      for (let i = 0; i < links_length; i++) {
        const link = $(links[i]).attr("href");
        this.links.push(link);
      }

      //
      // Load other page
      axios
        .get(this.links[1], {
          method: "get",
          redirect: "follow"
        })
        .then(res => {
          console.log("Cargando página...");
          $ = cheerio.load(res.data);
          const video_tabs = $("#player .tabs-video ul");
          //
          // Primer enlace
          const data = $(video_tabs[0])
            .find("li")
            .data();
          this.episodesCodes.push(data);
        })
        .catch(err => (this.error = err));

      // console.log($(video_tabs.find("li").data()));

      // axios
      //   .get(33620")
      //   .then(res => (this.episode = res))
      //   .catch(err => (this.error = err));
    }
  }
};
</script>
