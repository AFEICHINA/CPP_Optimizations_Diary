# If you are using std::list<>, you are doing it wrong

<p align="center"><img src="linked_list.png" width="350"></p>

I am not wasting time here to repeat benchmark that a lot of people did already.

- [std::vector vs std::list benchmark](https://baptiste-wicht.com/posts/2012/11/cpp-benchmark-vector-vs-list.html)

- [Are lists evil? Bjarne Stroustrup](https://isocpp.org/blog/2014/06/stroustrup-lists)

- [Video from Bjarne Stroustrup keynote](https://www.youtube.com/watch?v=YQs6IC-vgmo)

You think you case is special. It is not. Just try using `std::vector<>`
 (with `reserve` if you can) and you will see.
 
When you have to `push_front` and `pop_front`, your default data structure should be 
 [std::deque<>](https://es.cppreference.com/w/cpp/container/deque). 
 
If you like very exotic alternatives (that you won't actually need most of the tile), have a look
at [plf::colony](https://plflib.org/colony.htm).
 
But seriously, just use `vector`or `deque`.




