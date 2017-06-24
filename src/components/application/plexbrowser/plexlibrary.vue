<template>
    <span>
        <span v-on:click="reset()" style="cursor: pointer !important"> {{ library.title }} <span v-if="browsingContent"> > </span></span>        
        <v-layout v-if="!contents && shadowItems.length == 0" row>
            <v-flex xs12 style="position:relative">
                <v-progress-circular style="left: 50%; top:50%" v-bind:size="60" indeterminate class="amber--text"></v-progress-circular>
            </v-flex>
        </v-layout>
        <div v-if="!browsingContent && shadowItems.length > 0" class="mt-3">
          <v-layout class="row" row wrap>
              <v-flex xs4 md3 xl1 lg2  class="pb-3" v-for="(content, index) in shadowItems" :key="content">
                <plexthumb :index="index" :content="content" :server="server" type="thumb"  @contentSet="setContent(content)" @amVisible="wantContent(index)"></plexthumb>
              </v-flex>
          </v-layout>   
        </div>        
        <plexalbum v-if="browsingContent && browsingContent.type == 'album'" :content="browsingContent" :server="server"
                    :library="library"></plexalbum>
        <plexartist v-if="browsingContent && browsingContent.type == 'artist'" :content="browsingContent" :server="server"
                    :library="library"></plexartist>
        <plexcontent v-if="browsingContent && (browsingContent.type == 'movie' || browsingContent.type == 'series') " :content="browsingContent"
                     :server="server" :library="library"></plexcontent>
        <plexseries v-if="browsingContent && browsingContent.type == 'show'" :content="browsingContent" :server="server"
                    :library="library"></plexseries>        

    </span>
</template>

<script>
  import plexcontent from './plexcontent'
  import plexseries from './plexseries'  
  import plexalbum from './plexalbum'  
  import plexthumb from './plexthumb'
  import plexartist from './plexartist'

  var _ = require('lodash');
  export default {
    props: ['library', 'server'],
    components: {
      plexcontent,
      plexseries,
      plexthumb,
      plexalbum,
      plexartist
    },
    created () {
      // Hit the PMS endpoing /library/sections
      var that = this
    },
    data () {
      return {
        browsingContent: null,
        startingIndex: 0,
        size: 100,
        libraryTotalSize: false,

        stopNewContent: false,
        busy: false,
        contents: null,
        status: "loading..",
        searchPhrase: null,

        shadowItems: [],
        wantedItems: {},
      }
    },
    mounted () {
      this.getMoreContent(0)
    },
    beforeDestroy () {

    },
    computed: {},
    methods: {
      setContent (content) {
        this.browsingContent = content
      },
      handler (component) {
      },
      getThumb (object) {
        var w = Math.round(Math.max(document.documentElement.clientWidth, window.innerWidth || 0));
        var h = Math.round(Math.max(document.documentElement.clientHeight, window.innerHeight || 0));
        return this.server.getUrlForLibraryLoc(object.thumb, w / 6, h / 4)
      },
      setBackground () {        
        var w = Math.round(Math.max(document.documentElement.clientWidth, window.innerWidth || 0));
        var h = Math.round(Math.max(document.documentElement.clientHeight, window.innerHeight || 0));

        let randomItem = _.sample(this.contents.MediaContainer.Metadata)
        let url = randomItem.thumb 
        if (randomItem.type == 'show') {
          url = randomItem.art
        }
        this.$store.commit('SET_BACKGROUND',this.server.getUrlForLibraryLoc(url, w / 4, h / 4, 8))
      },
      isShown (item) {
        if (!item.active) {
          return {
            display: 'none'
          }
        }
        return {}
      },
      getTitleMovie(movie){
        if (movie.year){
          return movie.title + ' (' + movie.year + ')'
        }
        return movie.title
      },
      reset () {
        this.browsingContent = false
        this.setBackground()
      },
      wantContent: function (index) {
        console.log('Wanted: ' + index)
        this.wantedItems[index] = true
        this.getMoreContent()
      },
      getMoreContent: function(){
         if (this.stopNewContent || this.busy) {
          return
        }
        let index = 99999999
        for (let i in this.wantedItems){
          if (i < index){
            index = i
          }
        }
        if (index == 99999999){
          index = 0
          if (this.shadowItems[0]){
            return
          }
        }
        var that = this
        this.busy = true
        console.log('We need to get more content!')
        console.log('Searchin at index: ' + index)
        this.server.getLibraryContents(this.library.key, index, 100, (result) => {
          console.log('Metadata result', result)
          if (result && result.MediaContainer && result.MediaContainer.Metadata) {              
            const startIndex = result.MediaContainer.offset
            if (this.shadowItems.length == 0){
              // First result
              this.shadowItems = _.times(result.MediaContainer.totalSize, _.constant(null));
            }                 
            for (let i = 0; i < result.MediaContainer.Metadata.length; i++){
              //console.log('Setting: ' + (startIndex + i))
              this.shadowItems.splice(parseInt(startIndex + i), 1, result.MediaContainer.Metadata[i])
              if (this.wantedItems[parseInt(startIndex + i)]){
                delete this.wantedItems[parseInt(startIndex + i)]
              }
            }
          } else {
            that.status = 'Error loading libraries!'
          }
          that.busy = false
          this.getMoreContent()
        })
      }
    }

    
  }
</script>
