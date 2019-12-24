<template lang="pug">
  .root
    h1 {{ url }}
    video(ref="refVideo" width="100%" controls)
    .controls(v-show="optionsCurrentEpisode.resolutions[0].label !== null")
      button.btn(v-for="resolution in optionsCurrentEpisode.resolutions") {{ resolution.label }}
    div(v-for="temp in links")
      h3 {{ temp.season }}
      ul
        li.btn(v-for="episode in temp.episodes" @click="getVideoInformation(episode)") {{ episode.split("/").reverse()[0] }}
</template>

<script>
import cheerio from "cheerio";
import axios from "axios";

export default {
  data() {
    return {
      baseurl: "https://www.rexpelis.com/",
      tipo: "serie/",
      url: "12-monos",

      baseurlEpisodes: "https://www.rexpelis.com/player/embed/episode/",
      html: null,

      currentEpisode: {},

      optionsCurrentEpisode: {
        resolutions: [
          {
            label: null,
            src: null
          }
        ]
      },

      links: [
        {
          season: null,
          episodes: []
        }
      ],

      response: null,
      error: null
    };
  },

  mounted() {
    this.getWebSite();
  },

  watch: {
    response() {
      this.scrapy_serie(this.html);
    },

    currentEpisode(pValue) {
      const embedUrl = `${this.baseurlEpisodes}${pValue.playerId}`;

      // TODO: Reorganizar código para no tener múltiples llamadas de esta manera

      axios
        .get(embedUrl)
        .then(res => {
          const iframe = res.data;

          const $ = cheerio.load(iframe);

          const source = $(iframe).attr("src");
          const arrSecretCode = source.split("/");

          const urlApiSource = `https://${arrSecretCode[2]}/api/source/${arrSecretCode[4]}`;

          axios.post(urlApiSource).then(res => {
            this.optionsCurrentEpisode.resolutions = [];

            //
            // Standard Definition
            this.optionsCurrentEpisode.resolutions.push({
              label: "480 SD",
              src: res.data.data[0].file
            });

            //
            // Hight Definition
            if (res.data.data[1]) {
              this.optionsCurrentEpisode.resolutions.push({
                label: "720 HD",
                src: res.data.data[1].file
              });
            }

            // 1080 HD
            if (res.data.data[2]) {
              this.optionsCurrentEpisode.resolutions.push({
                label: "1080 HD",
                src: res.data.data[2].file
              });
            }

            //
            // Default loads SD
            this.$refs.refVideo.src = this.optionsCurrentEpisode.resolutions[0].src;
            this.$refs.refVideo.play();
          });
        })
        .catch(err => (this.error = err));
    }
  },

  methods: {
    /**
     * Hace una petición vía axios
     * para obtener la lista de títulos
     */
    getWebSite() {
      axios
        .get(`${this.baseurl}${this.tipo}${this.url}`, {
          method: "get",
          redirect: "follow"
        })
        .then(res => {
          this.response = res;
          this.html = res.data;
        })
        .catch(err => (this.error = err));
    },

    /**
     * Hace un rastreo de la página solicitada
     * y obtiene los enlaces de los capítulos de alguna serie.
     *
     * @param pHtml HTML que analizará los capítulos de la serie.
     */
    scrapy_serie(pHtml) {
      let $ = cheerio.load(pHtml);

      const episodes = $(".item-season-episodes");

      //
      // Encuentra los enlaces del array de las temporadas
      let links = [];
      this.links = [];

      const episodes_length = $(episodes).length;
      for (let i = 0; i < episodes_length; i++) {
        const link = $(episodes[i]).find("a");
        links.push(link);
      }

      for (let i = links.length - 1; i > 0; i--) {
        const URL = $(links[i]).attr("href");
        const TITLE = URL.split("/").reverse()[1];

        let episodes = [];

        for (let j = 0; j < links[i].length; j++) {
          const link = $(links[i][j]).attr("href");
          episodes.push(link);
        }

        this.links.push({
          season: TITLE,
          episodes
        });
      }
    },

    getVideoInformation(pLink) {
      axios
        .get(pLink, {
          method: "get",
          redirect: "follow"
        })
        .then(res => {
          let $ = cheerio.load(res.data);
          const video_tabs = $("#player .tabs-video ul");
          //
          // Primer enlace
          const data = $(video_tabs[0])
            .find("li")
            .data();

          //
          // Emit for current episode
          this.currentEpisode = data;
        })
        .catch(err => (this.error = err));
    }
  }
};
</script>

<style lang="scss" scoped>
.root {
  width: 100%;
  max-width: 700px;
  margin: auto;
}
ul {
  margin: 30px 0px;
  padding: 0;
}

li {
  list-style: none;
}

.btn {
  min-width: 100px;
  border: 1px solid;
  padding: 10px;
  margin: 15px 5px;
  text-align: center;
  background-color: #15161d;
  border-radius: 3px;
  color: rgba(5, 255, 255, 0.863);

  &:hover {
    cursor: pointer;
  }
}

.btn-active {
  background-color: rgb(0, 224, 176);
  color: black;
}
</style>
