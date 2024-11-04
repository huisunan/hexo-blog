---
title: 写一个通用的TreeUtil，自关联表结构的树形构造
date: 2022-03-18 10:35
tags: java
categories: 
---

<!--more-->

```java

package com.ccsa.common.core.util;

import lombok.experimental.UtilityClass;
import org.springframework.lang.Nullable;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

@UtilityClass
public class TreeUtil {


    public <T, I, V> List<V> treeList(List<T> originalList, I rootId, Function<T, I> getId, Function<T, I> getPid,
                                      Function<V, List<V>> getChildrenList, Function<T, V> handleV) {
        return treeList(originalList, rootId, getId, getPid, getChildrenList, handleV, null);
    }

    public <T, I, V> List<V> treeList(List<T> originalList, I rootId, Function<T, I> getId, Function<T, I> getPid,
                                      Function<V, List<V>> getChildrenList, Function<T, V> handleV, @Nullable OtherHandle<T, I, V> otherHandle) {
        return tree(originalList, rootId, getId, getPid, getChildrenList, handleV, otherHandle).entrySet().stream()
                .filter(i -> rootId.equals(i.getKey())).map(Map.Entry::getValue).collect(Collectors.toList());
    }

    public <T, I, V> Map<I, V> tree(List<T> originalList, I rootId, Function<T, I> getId, Function<T, I> getPid,
                                    Function<V, List<V>> getChildrenList, Function<T, V> handleV) {
        return tree(originalList, rootId, getId, getPid, getChildrenList, handleV, null);
    }

    /**
     * treeUtil
     *
     * @param originalList    原始列表
     * @param rootId          根结点
     * @param getId           获取id
     * @param getPid          获取pid
     * @param getChildrenList 获取子节点的list
     * @param handleV         处理Vo,这里要初始化好 childrenList
     * @param otherHandle     其他处理
     * @param <T>             入参
     * @param <I>             id类型
     * @param <V>             出参
     * @return 结果树
     */
    public <T, I, V> Map<I, V> tree(List<T> originalList, I rootId, Function<T, I> getId, Function<T, I> getPid,
                                    Function<V, List<V>> getChildrenList, Function<T, V> handleV,
                                    @Nullable OtherHandle<T, I, V> otherHandle) {
        //构建map
        Map<I, V> resultMap = new LinkedHashMap<>();
        Map<I, T> map = originalList.stream().collect(Collectors.toMap(getId, Function.identity()));
        //
        for (T t : originalList) {
            I pid = getPid.apply(t);
            I id = getId.apply(t);
            V v = resultMap.get(id);
            if (v == null){
               v = handleV.apply(t);
            }

            if (pid == rootId) {
                //判断顶层是否存在,不存在则创建，处理没有子项的顶层
                if (!resultMap.containsKey(id)) {
                    resultMap.put(id, v);
                }
                continue;
            }
            //获取父类,将自己添加到父类
            V parentV = resultMap.computeIfAbsent(pid, key -> handleV.apply(map.get(key)));
            //将自己加入结果map
            resultMap.putIfAbsent(id, v);
            //父节点的子节点列表
            List<V> parentChildrenList = getChildrenList.apply(parentV);
            parentChildrenList.add(v);
            if (otherHandle != null) {
                otherHandle.accept(v, parentV, t, map, resultMap);
            }
        }
        return resultMap;
    }

    public interface OtherHandle<T, I, V> {
        /**
         * 其他处理
         *
         * @param vo        vo
         * @param parentVo  父vo
         * @param t         原始参数
         * @param cacheMap  原始缓存
         * @param resultMap 结果集缓存
         */
        void accept(V vo, V parentVo, T t, Map<I, T> cacheMap, Map<I, V> resultMap);
    }
}

```