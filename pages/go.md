---
layout: page
title: Go Core
titlebar: go core
subtitle: <span class="mega-octicon octicon-clippy"></span> &nbsp;&nbsp; Go语言是谷歌2009年发布的第二款开源编程语言,它专门针对多处理器系统应用程序的编程进行了优化，它是一种系统语言其非常有用和强大,其程序可以媲美C或C++代码的速度，而且更加安全、支持并行进程。
menu: go
css: ['blog-page.css']
permalink: /go
---

<div class="row">

    <div class="col-md-12">
    
        <ul id="posts-list">
            {% for post in site.posts %}
                {% if post.category=='go' or post.category=='go' or post.keywords contains 'go' %}
                <li class="posts-list-item">
                    <div class="posts-content">
                        <span class="posts-list-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
                        <a class="posts-list-name bubble-float-left" href="{{ site.url }}{{ post.url }}">{{ post.title }}</a>
                        <span class='circle'></span>
                    </div>
                </li>
                {% endif %}
            {% endfor %}
        </ul> 
    
        <!-- Pagination -->
        {% include pagination.html %}
    
        <!-- Comments -->
       <div class="comment">
         {% include comments.html %}
       </div>
    </div>

</div>
<script>
    $(document).ready(function(){

        // Enable bootstrap tooltip
        $("body").tooltip({ selector: '[data-toggle=tooltip]' });
    
    });
</script>