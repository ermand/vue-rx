<template>
  <section class="container">
    {{activeTab$}}
    <b-tabs v-model="activeTab">
      <b-tab-item v-for="person of people$" :key="person.id" :label="person.name"></b-tab-item>
    </b-tabs>
    <h1 class="title">{{ name$ }}</h1>
    <img v-stream:error="imageError$" :src="image$" alt>
  </section>
</template>

<script>
import { Observable } from "rxjs";

export default {
  data() {
    return {
      activeTab: 0
    };
  },
  domStreams: ["click$", "imageError$"],
  subscriptions() {
    const cache = {};
    const cachePerson = cache => url => {
      return cache[url] ? cache[url] : (cache[url] = createLoader(url));
    };
    const createLoader = url =>
      Observable.from(this.$http.get(url)).pluck("data");

    const people$ = createLoader(
      "https://starwars.egghead.training/people"
    ).map(people => people.slice(0, 7));

    const activeTab$ = this.$watchAsObservable("activeTab", {
      immediate: true
    }).pluck("newValue");

    const luke$ = activeTab$
      .combineLatest(people$, (tabId, people) => people[tabId].id)
      .map(id => `https://starwars.egghead.training/people/${id}`)
      .switchMap(cachePerson(cache))
      .catch(err => {
        console.log("error:", err); //eslint-disable-line
        createLoader("https://starwars.egghead.training/people/2");
      })
      .share();

    const disabled$ = Observable.merge(
      this.click$.mapTo(true),
      luke$.mapTo(false)
    ).startWith(false);

    const buttonText$ = disabled$.map(bool => (bool ? "Loading.." : "Load"));

    const name$ = luke$.pluck("name");
    const loadImage$ = luke$
      .pluck("image")
      .map(image => `https://starwars.egghead.training/${image}`);

    const failImage$ = this.imageError$.mapTo(
      `http://via.placeholder.com/400x400`
    );
    const image$ = Observable.merge(loadImage$, failImage$);

    return {
      name$,
      image$,
      disabled$,
      buttonText$,
      activeTab$,
      people$
    };
  }
};
</script>

<style>
.container {
  margin-top: 2em;
}
</style>
