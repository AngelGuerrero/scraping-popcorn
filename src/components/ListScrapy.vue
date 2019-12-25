<template lang="pug">
  .root
    h1 {{ getFormattedName( url ) }}

    //- pre {{ $data }}

    .video__wrapper
      header
        h3 {{ selectedEpisode.season }} {{ selectedEpisode.name }}
      video(v-show="currentEpisode" ref="refVideo" width="100%" controls)
      .controls(v-show="showOptions")
        button.btn(v-for="resolution in selectedEpisode.options.resolutions"
                  :class="[ resolution.label == selectedEpisode.resolution ? 'btn-active' : '' ]"
                  @click="changeResolutionType(resolution.label)"
                  ) {{ resolution.label }}

    aside.episodes__wrapper
      //- TODO: Este menú va arriba dentro del menú de navegación
      .episodes__menu
        header
          h2 Menú
      .episodes(v-for="temp in arr_links")
        h3.episodes__title {{ getFormattedName(temp.season) }}
        ul.episodes__items
          li.btn(v-for="episode in temp.episodes"
                @click="getVideoInformation(episode)") {{ getChaptername(episode) }}
</template>

<script>
import cheerio from "cheerio";
import axios from "axios";

export default {
  props: {
    ppUrl: {
      type: String,
      required: true,
      default: "12-monos"
    }
  },

  data() {
    return {
      baseurl: "https://www.rexpelis.com/",
      tipo: "serie/",
      url: this.ppUrl,

      baseurlEpisodes: "https://www.rexpelis.com/player/embed/episode/",
      html: null,

      currentEpisode: null,

      //
      // Selected Episode and its options
      selectedEpisode: {
        season: "",
        name: "",
        resolution: "",
        options: {
          resolutions: [
            {
              label: null,
              src: null
            }
          ]
        }
      },

      arr_links: [
        {
          season: null,
          episodes: []
        }
      ],

      response: null,

      error: null
    };
  },

  computed: {
    showOptions: function() {
      return this.selectedEpisode.options.resolutions[0].label !== null;
    },

    getFormattedName: function() {
      return pValue => {
        if (pValue == null) return;

        return (
          pValue
            .split("-")
            .join(" ")
            // Capitalize
            .replace(/^\w/, c => c.toUpperCase())
        );
      };
    },

    getChaptername: function() {
      return pValue => {
        if (pValue == null) return;

        return this.getFormattedName(pValue.split("/").reverse()[0]);
      };
    }
  },

  mounted() {
    this.getWebSite();
  },

  watch: {
    response() {
      this.scrapy_serie(this.html);
    },

    currentEpisode(pValue) {
      const EMBED_URL = `${this.baseurlEpisodes}${pValue.playerId}`;

      // TODO: Reorganizar código para no tener múltiples llamadas de esta manera

      axios
        .get(EMBED_URL)
        .then(res => {
          const IFRAME = res.data;

          const $ = cheerio.load(IFRAME);

          const ARR_SOURCE = $(IFRAME).attr("src");
          const ARR_SECRET_CODE = ARR_SOURCE.split("/");

          const URL = `https://${ARR_SECRET_CODE[2]}/api/source/${ARR_SECRET_CODE[4]}`;

          axios.post(URL).then(res => {
            this.selectedEpisode.options.resolutions = [];

            //
            // Standard Definition
            this.selectedEpisode.options.resolutions.push({
              label: "480 SD",
              src: res.data.data[0].file
            });

            //
            // Hight Definition
            if (res.data.data[1]) {
              this.selectedEpisode.options.resolutions.push({
                label: "720 HD",
                src: res.data.data[1].file
              });
            }

            // 1080 HD
            if (res.data.data[2]) {
              this.selectedEpisode.options.resolutions.push({
                label: "1080 HD",
                src: res.data.data[2].file
              });
            }

            //
            // Default loads SD
            this.selectedEpisode.resolution = this.selectedEpisode.options.resolutions[0].label;
            this.$refs.refVideo.src = this.selectedEpisode.options.resolutions[0].src;
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
      //
      // Clear the data
      this.arr_links = [];

      let $ = cheerio.load(pHtml);

      const HTML_NODE_EPISODES = $(".item-season-episodes");

      //
      // Encuentra los enlaces del array de las temporadas let arr_links = [];
      let accum_links = [];

      for (let i = 0; i < $(HTML_NODE_EPISODES).length; i++) {
        const LINK = $(HTML_NODE_EPISODES[i]).find("a");
        accum_links.push(LINK);
      }

      for (let i = accum_links.length - 1; i >= 0; i--) {
        const URL = $(accum_links[i]).attr("href");
        const TITLE = URL.split("/").reverse()[1];

        let accum_episodes = [];

        for (let j = 0; j < accum_links[i].length; j++) {
          const EPISODE = $(accum_links[i][j]).attr("href");
          accum_episodes.push(EPISODE);
        }

        //
        // Push the elements into the array arr_links data object
        this.arr_links.push({
          season: TITLE,
          episodes: accum_episodes
        });
      }
    },

    getVideoInformation(pLink) {
      axios
        .get(pLink)
        .then(res => {
          let $ = cheerio.load(res.data);
          const ARR_VIDEO_TABS = $("#player .tabs-video ul");
          //
          // Primer enlace
          const DATA_VIDEO = $(ARR_VIDEO_TABS[0])
            .find("li")
            .data();

          this.selectedEpisode.name = this.getChaptername(pLink);
          this.selectedEpisode.season = this.getFormattedName(
            pLink.split("/").reverse()[1]
          );

          //
          // Emit for current episode
          this.currentEpisode = DATA_VIDEO;
        })
        .catch(err => (this.error = err));
    },

    changeResolutionType(pType) {
      const ITEM = this.selectedEpisode.options.resolutions.find(
        item => item.label == pType
      );

      this.selectedEpisode.resolution = ITEM.label;
      this.$refs.refVideo.src = ITEM.src;
      this.$refs.refVideo.play();
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

.video__wrapper {
  video {
    box-shadow: 7px 7px 20px rgb(22, 22, 22);
  }
}

ul {
  margin: 30px 0px;
  padding: 0;
}

li {
  list-style: none;
}

$btn-background-color: #15161d;

.btn {
  min-width: 100px;
  border: 1px solid;
  padding: 10px;
  margin: 15px 5px;
  text-align: center;
  background-color: $btn-background-color;
  border-radius: 3px;
  color: rgba(5, 255, 255, 0.863);
  transition: all 0.5s;

  &:hover {
    cursor: pointer;
    background-color: rgb(66, 36, 66);
    color: white;
    border-color: rgba(5, 255, 255, 0.863);
  }
}

.btn-active {
  background-color: rgb(0, 224, 176);
  color: black;
  &:hover {
    background-color: rgb(0, 224, 176);
    color: black;
  }
}

.episodes__wrapper {
  width: 100%;
  max-width: 250px;
  min-height: 100vh;
  max-height: 100vh;
  overflow: auto;
  position: fixed;
  top: 0;
  left: 0;
  z-index: 1;
}

.episodes {
  background-color: #17151d;

  &__title {
    background-color: rgb(22, 22, 22);
    margin: 0;
    padding: 10px;
    display: flex;
    align-items: center;
  }

  &__items {
    display: block;
    margin: 0;
    .btn {
      margin: 0px;
      border: none;
    }
  }
}
</style>
