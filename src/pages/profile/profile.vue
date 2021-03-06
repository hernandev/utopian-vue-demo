<script>
import ULayoutPage from 'src/layouts/parts/page/page'
import UPostPreview from 'src/components/post-preview/post-preview'
import { byOrder } from 'src/services/steem/posts'
import { getFollowCount } from 'src/services/steem/account'
import { concat, last, attempt, get } from 'lodash-es'
import { mapGetters, mapActions } from 'vuex'

export default {
  name: 'PageProfile',
  components: {
    ULayoutPage,
    UPostPreview
  },
  data () {
    return {    
      // auth store getters.
      ...mapGetters('auth', [
        'uid',
        'account',
        'username',
        'avatar'
      ]),
      userProfile: {},
      followerCount: 0,
      followersCount: {},
      userAccount: {},
      contributions: [],
      isMounted: false,
      loading: false
    }
  },
  filters: {
  },
  methods: {
    ...mapActions({
      loadFirebaseAccount: 'auth/loadFirebaseAccount',
      loadSteemAccount: 'steem/loadAccount'
    }),
    factoryProfile (data) {
      // extract meta information.
      const meta = JSON.parse(get(data, 'json_metadata', '{}') || '{}')
      // extract profile from meta.
      const profile = get(meta, 'profile', {})
      // assign avatar field on the profile.
      profile.avatar = 'https://img.blocker.press/a/' + data.name
      // return the prepared object.
      return profile
    },
    async loadInitial () {
      const username = this.$route.params['username']

      if (this.isOwnProfile) { // if the profile belongs to the current logged user, fetch data from vuex
        this.userAccount = this.account()
        this.userProfile = this.userAccount.profile
        return this.userProfile
      } else { // if it is not the logged user, try to fetch from firestore
        this.loadFirebaseAccount(username).then((account) => {
          if (account === null) { // if the user is not registered on firestore, fetch from the blockchain
            this.loadSteemAccount(username).then((account) => {
              this.userAccount = account
              this.userProfile = this.factoryProfile(account)
              return this.userProfile
            })
          } else {
            this.userAccount = account
            this.userProfile = account.profile
            return this.userProfile
          }
        })
      }
    },
    loadContributions (done) {
      return byOrder('trending', { tag: 'utopian-io', limit: 10 }, last(this.posts))
        .then((result) => {
          this.contributions = concat(this.contributions, result)
          attempt(done)
          return result
        })
    },
    loadFollowCount () {
      return getFollowCount(this.$route.params['username']).then((result) => {
        this.followerCount = result.follower_count
        this.followingCount = result.following_count
      })
    }
  },
  computed: {
    isOwnProfile () {
      return this.uid() === this.$route.params['username']
    },
    coverImage () {
      if (this.isMounted) {
        if (this.userProfile.cover_image) {
          return 'https://steemitimages.com/2048x512/' + this.userProfile.cover_image
        } else {
          return 'https://source.unsplash.com/2048x512/?coding,computer,tech'
        }
      }
      return ''
    }
  },
  mounted () {
    this.loading = true
    Promise.all([this.loadInitial(), this.loadFollowCount()]).then(() => {
      this.loading = false
      this.isMounted = true
    })
  },
  watch: {
  }
}
</script>

<style lang="stylus" src="./profile.styl"></style>

<template lang="pug" src="./profile.pug"></template>
