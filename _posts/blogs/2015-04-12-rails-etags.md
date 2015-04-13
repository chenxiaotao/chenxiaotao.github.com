---
layout: post
categories: [blog, rails]
title: rails 中的Etags
tags : [rails, cache]
---
{% include JB/setup %}

## 1.什么是ETag
简单的说，ETag就是一种客户端缓存，是 HTTP 协议的标准参数，它能通过一段字符来判断浏览器 cache 的内容是否和服务端返回的内容是否相同，从而来决定是否要重新从服务器下载东西 (HTTP 状态 200 - 重新下载 / 304 - 没有更新)

eg:

* 首个请求：
	* 浏览器初始化首个请求
* 首个响应：
	* Rails生成响应内容
	* Rails生成ETag
	* Rails响应请求，响应信息中带有ETag头和状态码200 浏览器在接收到响应页面后，将缓存该页面。当浏览器在处理后续的请求的时，步骤如下：
* 后续请求
	* 浏览器发送头部信息带有'If-None-Matched'(存储的是Etag值)的请求
* 后续响应
	* Rails生成响应内容和ETag
	* 比较生成的ETag和请求中'If-None-Matched'字段的值
	* 若ETag相同，生成的响应内容将不返回浏览器，而将返回304状态码
	* 若ETag不相同，将返回新的ETag值和生成的响应内容

---

## 2.ETag的优势
那么，使用ETag机制来匹配服务器上的内容有什么好处呢？好处就是Rails将不发送生成的页面内容，这样响应体将变的更小从而使得其网络中的传输速度更快。浏览器通过加载自身缓存中的内容，使得网站刷新更快，体验更好。

---

## 3.Rails中的ETag
在Rails中，已经默认使用ETag机制，不需要额外操作，以下代码将自动使用Rails的默认ETag缓存机制

	class PostsController < ApplicationController
	  def show
		@post = Post. find(params[:id])

		respond_to do |format|
		  format.html # show.html.erb
		  format.json { render json: @post  }
		end
	  end

	  def edit
		@post = Post.find(params[:id])
	  end

	end
Rails生成响应内容，并根据生成的响应内容生成MD5 散列的ETag，类似下面：

	headers['ETag'] = Digest::MD5.hexdigest(body)
通过每次生成的响应内容来生成ETag并不能高效的利用服务器，因为这样服务器将耗时调用数据库和渲染模板文件。这时可以通过Rails的helper方法fresh_when和stale?来实现自定义的ETag

---

## 4.使用fresh_when和stale?实现自定义的ETag
* fresh_when

对于一个简单的文章页面 posts#show 页面，可以使用以下缓存

	class PostsController < ApplicationController
	  def show
		@post = Post.find(params[:id])
		fresh_when(@post)
	  end
	end
对于多个变量可能导致页面发生变化时, 只需要把会导致页面发生变化的变量加入 ETag 就行了，比如对于有若干回复列表的文章页面, 新增的回复也会引起页面的变化, 因此可以加入 ETag 计算

	def show
	  @post = Post.find(params[:id])
	  @replies = @post.replies
	  fresh_when([@post, @replies])
	end

* stale?

stale? 的参数跟 fresh_when 是一样的, 实际上 stale? 调用了 fresh_when 方法

	def stale?(record_or_options, additional_options = {})
	  fresh_when(record_or_options, additional_options)
	  !request.fresh?(response)
	end
stale? 返回一个布尔值, 如果此次请求需要正常渲染, 则返回 true, 如果是 304 无需渲染页面, 则返回 false。因此, 对于上面的文章页面, 可以更改为

	def show
	  @post = Post.find(params[:id])
	  if stale?(@post)
		@replies = @post.replies
	  end
	end

* 什么时候使用fresh_when和stale?方法，它们之间有什么区别？

若你有特定的响应处理（如下面代码中的show动作），请使用stale?方法；若你没有特定的响应处理，例如你不需要使用respond_to或调用render方法（如下面代码中的edit和recent动作），请使用fresh_when。

	class PostsController < ApplicationController
	  def show
		@post = Post. find(params[:id])
		if stale?(@post, current_user_id: current_user.id)
		  respond_to do |format|
			format.html # show.html.erb
			format.json { render json: @post  }
		  end
		end
	  end

	  def edit
		@post = Post.find(params[:id])
		fresh_when(@post, current_user_id: current_user.id)
	  end

	  def recent
		@post = Post.find(params[:id])
		fresh_when(@post, current_user_id: current_user.id)
	  end

	end

---

## 5.Rails 4中的声明式ETag特性

Rails 3 和Rails 4都默认使用ETags机制处理浏览器缓存，但Rails 4添加了声明式ETags特性，该特性允许你在控制器中添加全局的Etag信息。

	class PostsController < ApplicationController
	  etag { current_user.id  }
	  def show
		@post = Post. find(params[:id])
		if stale?(@post)
		  respond_to do |format|
			format.html # show.html.erb
			format.json { render json: @post  }
		  end
		end
	  end

	  def edit
		@post = Post.find(params[:id])
		fresh_when(@post)
	  end

	  def recent
		@post = Post.find(params[:id])
		fresh_when(@post)
	  end

	end

你可以生成多个ETags:

	class PostsController < ApplicationController
	  etag { current_user.id  }
	  etag { current_customer.id  }
	  # ...
	end

你也能够设置ETags的生成条件：

	class PostsController < ApplicationController
	  etag do
		{ current_user_id: current_user.id  } if %w(show edit).include? params[:action]
	  end
	end
