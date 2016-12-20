---
layout: post
cover: 'assets/images/cover1.jpg'
title: 레일즈 Scope을 사전 적재하기
date:   2016-11-25 00:00:00
tags: rails
subclass: 'post tag-fiction'
categories: 'npmachine'
navigation: True
---

> Justin Weiss의 [How to Preload Rails Scopes](http://www.justinweiss.com/articles/how-to-preload-rails-scopes/)를 번역한 글입니다.

---

레일즈 Scope은 다음과 같이 원하는 레코드를 쉽게 찾도록 도와준다.

```ruby
class Review < ActiveRecord::Base
  belongs_to :restaurant

  scope :positive, -> { where('rating > 3.0') }
end
```
```ruby
irb(main):001:0> Restaurant.first.reviews.positive.count
  Restaurant Load (0.4ms)  SELECT  'restaurants'.* FROM 'restaurants'  ORDER BY 'restaurants'.'id' ASC LIMIT 1
   (0.6ms)  SELECT COUNT(*) FROM 'reviews' WHERE 'reviews'.'restaurant_id' = 1 AND (rating > 3.0)
=> 5
```

그렇지만 주의깊게 사용하지 않는다면 심각한 속도 저하 문제에 직면한다.
**Scope은 사전 적재(preload)할 수 없기 때문이다.** 예를 들어 좋은 리뷰를 받은 레스토랑 목록을 출력한다고 가정해보자.

```ruby
irb(main):001:0> restauraunts = Restaurant.first(5)
irb(main):002:0> restauraunts.map do |restaurant|
irb(main):003:1*   '#{restaurant.name}: #{restaurant.reviews.positive.length} positive reviews.'
irb(main):004:1> end
  Review Load (0.6ms)  SELECT 'reviews'.* FROM 'reviews' WHERE 'reviews'.'restaurant_id' = 1 AND (rating > 3.0)
  Review Load (0.5ms)  SELECT 'reviews'.* FROM 'reviews' WHERE 'reviews'.'restaurant_id' = 2 AND (rating > 3.0)
  Review Load (0.7ms)  SELECT 'reviews'.* FROM 'reviews' WHERE 'reviews'.'restaurant_id' = 3 AND (rating > 3.0)
  Review Load (0.7ms)  SELECT 'reviews'.* FROM 'reviews' WHERE 'reviews'.'restaurant_id' = 4 AND (rating > 3.0)
  Review Load (0.7ms)  SELECT 'reviews'.* FROM 'reviews' WHERE 'reviews'.'restaurant_id' = 5 AND (rating > 3.0)
=> ["Judd's Pub: 5 positive reviews.", "Felix's Nightclub: 6 positive reviews.", "Mabel's Burrito Shack: 7 positive reviews.", "Kendall's Burrito Shack: 2 positive reviews.", "Elisabeth\'s Deli: 15 positive reviews."]
```

위는 레일즈 앱을 느리게 만드는 가장 큰 주범인 [N + 1](http://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations) 쿼리 상황이다.

**이 문제는 관계를 다른 방법으로 설정하여 간단히 해결할 수 있다.**

### Scope을 association으로 변경하자  

`belongs_to`나 `has_many` 같은 레일즈 `association` 메소드는 보통 다음처럼 구현한다.

```ruby
class Restaurant < ActiveRecord::Base
  has_many :reviews
end
```
그렇지만 [레일즈 가이드 문서](http://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html)를 살펴보면, 더 많은 기능이 있다. 파라미터를 전달하여 동작 방식을 변경할 수도 있다.

그 중에서 가장 유용한 기능 중 하나가 `scope`이다. 앞서 살펴보았던 `scope`처럼 동작한다.

```ruby
class Restaurant < ActiveRecord::Base
  has_many :reviews
  has_many :positive_reviews, -> { where('rating > 3.0') }, class_name: 'Review'
end
```

```ruby
irb(main):001:0> Restaurant.first.positive_reviews.count
  Restaurant Load (0.2ms)  SELECT  'restaurants'.* FROM 'restaurants'  ORDER BY 'restaurants'.'id' ASC LIMIT 1
   (0.4ms)  SELECT COUNT(*) FROM 'reviews' WHERE 'reviews'.'restaurant_id' = 1 AND (rating > 3.0)
=> 5
```

이제 관계된 모델을 위에서처럼 `includes`로 사전 적재한다.

```ruby
irb(main):001:0> restauraunts = Restaurant.includes(:positive_reviews).first(5)
  Restaurant Load (0.3ms)  SELECT  'restaurants'.* FROM 'restaurants'  ORDER BY 'restaurants'.'id' ASC LIMIT 5
  Review Load (1.2ms)  SELECT 'reviews'.* FROM 'reviews' WHERE (rating > 3.0) AND 'reviews'.'restaurant_id' IN (1, 2, 3, 4, 5)
irb(main):002:0> restauraunts.map do |restaurant|
irb(main):003:1*   '#{restaurant.name}: #{restaurant.positive_reviews.length} positive reviews.'
irb(main):004:1> end
=> ["Judd's Pub: 5 positive reviews.", "Felix's Nightclub: 6 positive reviews.", "Mabel's Burrito Shack: 7 positive reviews.", "Kendall's Burrito Shack: 2 positive reviews.", "Elisabeth's Deli: 15 positive reviews."]
```

6개의 SQL문 호출이 2개로 줄었다.
(`class_name`을 이용하면, 하나의 객체로 여러 개의 관계를 맺을 수 있다. 이는 매우 편리하다.)

### 코드 중복은 어떻게 처리할까?  

  

아직 잠재적인 문제가 남아있다. Restaurant 클래스에는 `where('rating > 3.0')` 조건이 있다. 나중에 우수 리뷰 조건을 `rating > 3.5`로 변경해야한다면, 두 번 변경해야 한다!

더 나쁜 경우도 있다. 우수한 리뷰를 고객별로 검색할 수 있는 기능을 추가할 경우에는 User 클래스에도 중복 코드를 추가해야 한다.

```ruby
class User < ActiveRecord::Base
  has_many :reviews
  has_many :positive_reviews, -> { where('rating > 3.0') }, class_name: 'Review'
end
```

이는 전혀 [DRY](http://www.c2.com/cgi/wiki?DontRepeatYourself)하지 않다.  
다행히 해결법은 간단하다. Review 클래스에 `where` 대신 **`positive scope`을 추가하면 된다.**

```ruby
class Restaurant < ActiveRecord::Base
  has_many :reviews
  has_many :positive_reviews, -> { positive }, class_name: "Review"
end
```

우수 리뷰의 조건 코드를 마침내 한 곳으로 모으게 되었다.

---

Scope은 무척 유용하다. 적절히 사용한다면, 쿼리를 쉽고 재미있게 만들 수 있다. 그렇지만 N + 1 쿼리 이슈를 피하고자 한다면, 주의해서 사용해야 한다.

**그러므로, scope이 문제가 된다면, association으로 추상화하고 사전 적재하라.** 적은 추가 작업으로도 많은 SQL 호출을 줄일 수 있다.
