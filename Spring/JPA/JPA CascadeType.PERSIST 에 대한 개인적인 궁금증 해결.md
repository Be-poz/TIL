# JPA CascadeType.PERSIST 에 대한 개인적인 궁금증 해결

Team과 Member와 같은 1:n의 연관관계를 처리할 때에 Team에 Member를 넣은 후 일일이 Member를 영속화하기 번거로우니 Team만 영속화시켜도 Member 또한 영속화되도록 하기위해 설정을 해준다. 이렇듯이 보통 1에서 n에 대하여 이 속성을 걸어주게된다.  

```java
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "team", cascade = CascadeType.PERSIST)
    private List<Member> members = new ArrayList<>();
    
    public Team(String name) {
        this.name = name;
    }

    public void addMember(Member member) {
        members.add(member);
      	member.setTeam(this);
    }
}

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;

    public Member(String name) {
        this.name = name;
    }
}
```

위와 같이 1:n의 관계로 Team과 Member가 존재한다.  

```java
    @Test
    public void cascadeTest() {
        Team team1 = new Team("team1");
        Member member1 = new Member("member1");
        team1.addMember(member1);
        em.persist(team1);
        em.flush();
        em.clear();
    }
```

이 상황에서 위의 코드를 수행하면 Team Insert쿼리가 나오고 그 후 team_id를 가진 상태인 Member의 Insert 쿼리가 나가게 된다. 그렇다면, CascadeType.PERSIST를 걸어준 상태에서는 Team의 ``List<Member> members``에 자식인 Member를 걸어주기만 하면 알아서 Member가 해당 Team의 id를 알고 그것까지 쿼리문에 반영해주는지 궁금해졌다.  

```java
    public void addMember(Member member) {
        members.add(member);
        //member.setTeam(this);
    }
```

연관관계의 주인인 Member한테 Team을 걸어주는 로직을 주석처리하고 진행해보았다. 그 결과는 team_id가 null인채로 들어가는 것을 확인할 수가 있었다.  

그렇다면 1쪽이 아니라 n쪽에 건다면 어떻게 될까 . . ?  

```java
//Team.class
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();

//Member.class
    @ManyToOne(fetch = FetchType.LAZY, cascade = CascadeType.PERSIST)
    @JoinColumn(name = "team_id")
    private Team team;

//Test
    @Test
    public void cascadeTest() {
        Team team1 = new Team("team1");
        Member member1 = new Member("member1");
        member1.setTeam(team1);
        em.persist(member1);
        em.flush();
        em.clear();
    }
```

결과는 Team이 먼저 Insert된 후 Member에 team_id가 담긴채로 Insert문이 나가는 것을 확인할 수가 있었다. 둘 다 영속화 상태가 되어있는 상태에서 어떤쪽이 먼저 쿼리를 날릴지 궁금했는데 Team이 먼저 날리는 것이 굉장히 흥미로웠다. 그도 그럴 것이 Team이 먼저 날리지 않게된다면 team_id 없이 Member를 Insert하게 되고 추후에 team_id 세팅을 위해 update문을 날려야 하니 이렇게 내부적으로 n이 아닌 1부터 쿼리를 날리게 해둔 것 같다.  

그렇다면 n:n 이어서 중간 테이블을 둔 경우에는 어떻게 될까 ? ? Team이 영속화 될 때, 영속성 전이로 중간 테이블을 영속화하게 만들어 놓고, 이 중간 테이블에서도 Member에 대해서 CascadeType.PERSIST를 걸어놓은 경우에는 어떻게 될지 궁금해졌다.  

```java
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "team", cascade = CascadeType.PERSIST)
    private List<TeamMember> teamMembers = new ArrayList<>();

    public Team(String name) {
        this.name = name;
    }

    public void addMember(TeamMember teamMember) {
        this.teamMembers.add(teamMember);
        teamMember.setTeam(this);
    }
}

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class TeamMember {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;

    @ManyToOne(fetch = FetchType.LAZY, cascade = CascadeType.PERSIST)
    @JoinColumn(name = "member_id")
    private Member member;
}

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "member")
    private List<TeamMember> teamMembers = new ArrayList<>();

    public Member(String name) {
        this.name = name;
    }
}
```

Team과 Member가 n:n이기에 TeamMember라는 중간테이블을 두었고, 먼저 TeamMember과 Team, Member는 모두 n:1의 관계를 맺고있다.  

```java
    @Test
    public void cascadeTest() {
        Team team = new Team("team");
        Member member = new Member("member1");
        TeamMember teamMember = new TeamMember();
        teamMember.setMember(member);
        team.addMember(teamMember);

        em.persist(team);
        em.flush();
        em.clear();
    }
```

다음과 같은 코드를 실행해보니, Team, Member TeamMember 순으로 Insert 쿼리를 날리는 것을 확인할 수가 있었다.  
그저 중간 테이블의 역할만 가지고 있는 TeamMember이기에 위의 코드는 조금 복잡하므로 다음과 같이 리팩토링 할 수 있을 것 같다.  

```java
    @Test
    public void cascadeTest() {
        Team team = new Team("team");
        Member member = new Member("member1");
        team.addMember(member);

        em.persist(team);
        em.flush();
        em.clear();
    }

    public void addMember(Member member) {
        TeamMember teamMember = new TeamMember();
        teamMember.setMember(member);
        this.teamMembers.add(teamMember);
      	member.getTeamMembers().add(teamMember);
        teamMember.setTeam(this);
    }
```

위와 같이 수정 후 테스트 코드를 돌려보니 동일한 결과가 나오는 것을 확인할 수가 있었다.  

이번에는 Team : TeamMember : Member 각 n:1:n 으로 수정해서 수행해보겠다. 앞선 경우와 이 경우와의 차이점은 외래키를 누가 관리하고 있을 것이냐가 되겠다.  

```java
@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY, cascade = CascadeType.PERSIST)
    private TeamMember teamMember;

    public Team(String name) {
        this.name = name;
    }
}

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class TeamMember {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "teamMember")
    private List<Team> teams = new ArrayList<>();

    @OneToMany(mappedBy = "teamMember", cascade = CascadeType.PERSIST)
    private List<Member> member = new ArrayList<>();
}

@Entity
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private TeamMember teamMember;

    public Member(String name) {
        this.name = name;
    }
}
```

```java
    @Test
    public void cascadeTest() {
        Team team = new Team("team");
        Member member = new Member("member");
        TeamMember teamMember = new TeamMember();
        teamMember.getMember().add(member);
        team.setTeamMember(teamMember);

        em.persist(team);
        em.flush();
        em.clear();
    }
```

이번에는 TeamMember가 1의 상황이다 보니 TeamMember가 먼저 Insert 된 후 teamMember_id의 값을 가진 채로 Team과 Member가 Insert 되는 것을 확인할 수가 있었다. 위의 코드 또한 리팩토링을 해보겠다.  

```java
// Team.class
    public void addMember(Member member) {
        TeamMember teamMember = new TeamMember();
        teamMember.addMember(member);
        this.teamMember = teamMember;
    }

// TeamMember.class
    public void addMember(Member member) {
        this.members.add(member);
        member.setTeamMember(this);
    }
```

이렇게 확인해보았다. 이런 상황에서도 CascadeType.PERSIST를 걸어주는 것이 정말 맞는 것인지에 대한 확답을 내리기에는 아직 내 경험이 그렇게 많지 않아서 쉽지 않은 것 같다. 어쨋든 중간 테이블을 두는 상황에서도 cascade를 이용할 수 있다는 점을 확인해보았다!  

중간 테이블에서 생길 수 있는 주안점은 중간 테이블을 일대다로 가질 것인지 다대일로 가질 것인지에 대한 것이 있을 수 있겠다.  
외래키를 한 테이블에서 관리할 것인지 각 양쪽 엔티티에서 따로 관리할 것인지에 대한 차이점인데, 일단 나는 한 곳에서 관리하는 것이 조금 더 낫지않나? 라고 생각해본다. 위의 리팩토링 과정에서도 느꼈지만, 연관 관계를 걸어주는 코드 또한 흩어지지 않고 캡슐화가 더 되는 느낌이기도 하고 유지보수적으로 조금 더 편할 것 같다고 생각한다. 개인적인 의견일 뿐이다.  

***