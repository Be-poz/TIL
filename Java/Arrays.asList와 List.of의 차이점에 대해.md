# Arrays.asList와 List.of의 차이점에 대해

Arrays.asList와 List.of 둘 다 고정된 크기의 리스트를 제공하기 때문에 새 요소를 추가하거나 삭제하려고 하면 ``UnsupportedOperationException``이 발생한다. 그렇다면 둘의 차이점은 무엇일까 ??  

1. List.of 는 set으로 값 변경이 불가능하다.

   ```java
   List<Integer> asList = Arrays.asList(1, 2, 3);
   List<Integer> listOf = List.of(1, 2, 3);
   
   asList.set(0, 10);
   listOf.set(0, 10);		//UnsupportedOperationException
   ```

   List.of는 set으로 값 변경을 시도하면 컴파일 에러가 발생하게 된다.

2. List.of 는 null을 허용하지 않는다.

   ```java
   List<Integer> asList = Arrays.asList(1, 2, null);
   List<Integer> listOf = List.of(1, 2, null);		//NPE
   ```

   null을 받아들이는 Arrays.asList와 달리 List.of는 거부한다!!

3. List.of 는 null 여부를 contains 확인도 못하게 한다.

   ```java
   List<Integer> asList = Arrays.asList(1, 2, 3);
   List<Integer> listOf = List.of(1, 2, 3);
   
   boolean asListResult = asList.contains(null);
   boolean listOfResult = listOf.contains(null);		//NPE
   ```

   까다로운 녀석이다...

4. Arrays.asList는 원본의 배열의 변화에 반응한다.

   ```java
   Integer[] arr = {1, 2, 3};
   
   List<Integer> asList = Arrays.asList(arr);
   List<Integer> listOf = List.of(arr);
   
   arr[0] = 10;
   
   System.out.println(asList);
   System.out.println(listOf);
   /*
   [10, 2, 3]
   [1, 2, 3]
    */
   ```

   arr의 값이 변하자 asList의 값 또한 변한 것을 확인할 수가 있다.

<br/>

이렇게 변화를 살펴봤는데, 만약 ``new ArrayList<>(List.of / asList);`` 의 방식을 사용하는 경우에는,  
어떤 것을 사용하는지는 취향차이라고 생각한다.

***

### REFERENCE

https://stackoverflow.com/questions/46579074/what-is-the-difference-between-list-of-and-arrays-aslist
