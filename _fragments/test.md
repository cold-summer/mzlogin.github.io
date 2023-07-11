---
layout: fragment
title: test
tags: [test]
description: test文章
keywords: test
---

 public static void main(String[] args) {
        Thread[] threads = new Thread[2];
        threads[0] = new Thread(() -> {
            for (int i = 0; i < 50; ++i) {
                System.out.println("A");
                LockSupport.unpark(threads[1]);
                LockSupport.park();
            }
        });

        threads[1] = new Thread(() -> {
            for (int i = 0; i < 50; ++i) {
                LockSupport.park();
                System.out.println("B");
                LockSupport.unpark(threads[0]);

            }
        });
        threads[0].start();
        threads[1].start();
    }